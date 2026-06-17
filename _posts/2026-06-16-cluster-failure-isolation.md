---
title: "Ceph 클러스터 장애 전파를 차단한 서버 고가용성 구조 개선"
description: "Ceph 클러스터 장애가 서비스 전체로 확산되는 구조를 책임 분리(HAProxy, Resilience4j)로 끊고, 커스텀 메트릭으로 장애 모니터링 시스템을 구축한 과정."
date: 2026-06-16 12:00:00 +0900
categories: [Project, Architecture]
tags: [ceph, resilience4j, haproxy, circuit-breaker, prometheus, observability]
toc: true
---

## 프로젝트 개요
Ceph MGR API를 활용해 클러스터 상태, 스토리지 리소스, 장애 정보를 조회하고 이를 FE에 제공하는 Backend를 개발·운영했습니다.

기존 구조에서는 Backend가 Ceph MGR API를 직접 호출했고, Active MGR 변경에 따른 303 Redirect, fallback, retry, Active MGR host 갱신까지 애플리케이션 내부에서 처리하고 있었습니다. 평상시에는 문제가 크지 않았지만, Ceph 클러스터 장애 상황에서는 이 구조가 Backend API 지연으로 이어졌고, 그 영향이 FE까지 전파됐습니다.

이 문제를 단순한 예외 처리 코드를 늘려 해결할 문제가 아니라, **장애 처리 책임이 Backend 요청 흐름 안에 포함된 구조적 문제**로 정의했습니다. 이후 Ceph 장애가 Backend로 직접 전파되지 않도록 HAProxy 기반 진입점 분리, Resilience4j 기반 장애 격리, Prometheus 커스텀 메트릭 기반 장애 감지 구조로 개선했습니다.


| 항목 | 내용 |
|------|------|
| 기간 | 2025.10 ~ 2026.03 |
| 역할 | Backend 개발, 문제 정의, 아키텍처 재설계, 구현, 부하 테스트, 모니터링 구축 |
| 기술 | Java, Spring Boot, Resilience4j, HAProxy/Keepalived, Redis, Prometheus, Grafana, Alertmanager, k6 |
| 핵심 성과 | Ceph 장애 전파 차단, 장애 클러스터 fast-fail, Provider별 CircuitBreaker 격리, 커스텀 메트릭 기반 장애 감지 |


## 핵심 성과

| 항목 | 개선 전 | 개선 후 |
|------|--------|--------|
| 평균 응답 시간 | 5.04s | 36.5ms |
| P95 레이턴시 | 10.03s | 78ms |
| dropped_iterations | 989 | 11 |
| 장애 클러스터 fast-fail | 불가 | 98~99% |
| 장애 감지 | 수동 확인 | 메트릭 + Slack 알림 |

---

## 문제 상황

기존 요청 흐름은 다음과 같았습니다.

![기존 구조 시퀀스 다이어그램](/assets/img/ceph-seq-before.png)
_기존 구조 — Backend가 303 Redirect 파싱, Host 순회 Fallback, 재시도, Active MGR 갱신을 직접 처리했습니다._

Ceph MGR는 Active-Standby 구조로 동작합니다. 
Active MGR가 변경되거나 Standby MGR로 요청이 전달되면 `303 Redirect`가 발생할 수 있습니다. 기존 Backend는 이 과정에서 필요한 redirect 처리, fallback, retry, Active MGR host 갱신을 모두 직접 담당하고 있었습니다.

장애 상황에서는 이 구조가 다음 문제로 이어졌습니다.

- Ceph 클러스터 장애 시 Backend 평균 응답 시간이 최대 1.15초까지 증가했고, 지연이 FE까지 전파됐습니다.
- `redirect → retry → host 순회 → 재호출` 흐름이 중첩되면서 장애 원인이 Ceph, 인프라, Backend 중 어디에 있는지 구분하기 어려웠습니다.
- Ceph 연동 예외 처리 로직이 Backend 내부에 계속 추가되면서 코드 복잡도와 유지보수 비용이 증가했습니다.

핵심 문제는 Ceph 장애 자체가 아니라, **Ceph의 failover 처리와 장애 대응 책임이 Backend 요청 처리 흐름 안에 포함되어 있었다는 점**이었습니다.

---
## 1단계: 측정 가능한 상태 만들기 (Single-flight)
구조를 바꾸기 전에 먼저 Ceph 클러스터 장애 상황을 재현하고, 기존 구조가 실제로 어떻게 동작하는지 측정하려 했습니다. 하지만 부하 테스트 과정에서 Backend 내부의 동시성 문제가 먼저 드러났습니다.

동일 Provider에 요청이 몰릴 때 인증 토큰 발급 API가 중복 호출되었고, 토큰 캐시 갱신과 Active MGR 갱신 이벤트가 동시에 발생하면서 DB Optimistic Lock 충돌이 발생했습니다. 이 상태에서는 HAProxy나 Resilience4j를 적용해도 개선 효과를 신뢰하기 어려웠습니다.

처음에는 `synchronized`, `ReentrantLock`, `Redis 분산 락`을 검토했습니다. 하지만 락 내부에서 외부 HTTP 호출이 발생하는 구조였기 때문에, 락 범위가 길어질수록 병목이 커질 수 있었습니다.

Redis 분산 락도 검토했지만, 모든 인스턴스 간에 완전한 상호배제가 꼭 필요한 상황은 아니라고 판단했습니다. 토큰 발급 API는 클러스터 상태를 변경하지 않는 인증 요청이었고, 최악의 경우에도 중복 호출은 인스턴스 수만큼으로 제한됩니다. 따라서 분산 락으로 복잡도를 높이기보다, 인스턴스 내부에서 동일 Provider의 동시 요청만 합치는 Single-flight 방식을 선택했습니다.

```text
getOrIssueToken()
├─ Redis 캐시 조회 (Cache Hit 시 즉시 반환)
├─ cache miss
│
├─ inFlight.computeIfAbsent(providerId)
│   └─ 동일 providerId의 첫 요청만 Ceph /api/auth 호출
│
├─ 다른 요청은 동일 Future 결과 공유
└─ 결과 반환
```
이를 통해 토큰 발급 중복 호출을 줄이고, Active MGR 갱신 이벤트 중복 발생과 DB Lock 경합을 완화했습니다. 결과적으로 이후 구조 개선 효과를 신뢰할 수 있는 테스트 환경을 만들 수 있었습니다.

---

## 2단계: 인프라 책임 분리 (HAProxy + VIP)

Single-flight를 통해 Backend 내부 동시성 문제를 완화한 뒤, 기존 Ceph 연동 로직이 장애 상황에서 얼마나 큰 지연을 만드는지 측정했습니다. 동일한 Ceph 클러스터 장애 상황에서 기존 로직과 단순 호출 구조를 비교했습니다.

| 구분 | 평균 응답 시간 |
|------|-------------|
| 기존 로직 (redirect → retry → host 순회) | 1,150ms |
| redirect/fallback 제거 후 단순 호출 | 7.2ms |

측정 결과, 기존 redirect/fallback 로직은 장애 상황에서 평균 응답 시간을 크게 증가시키고 있었습니다. 이 결과를 통해 문제를 “재시도를 어떻게 잘 구현할 것인가”가 아니라, **Active MGR 선택과 failover 책임을 Backend가 가져야 하는가**라는 관점에서 다시 보게 되었습니다.

기존 Backend는 Active MGR 선택, Standby 판단, 장애 감지, Host 순회 Fallback을 모두 직접 수행했습니다. 이 구조에서는 장애가 발생할 때 Backend 내부 로직이 복잡하게 얽히고, 장애 원인이 Ceph인지, 인프라인지, Backend 로직인지 분리하기 어려웠습니다.

따라서 Backend가 더 이상 MGR 선택과 장애 판단을 직접 수행하지 않도록, Ceph MGR API 앞단에 Reverse Proxy를 두고 Backend는 항상 단일 진입점인 VIP만 호출하도록 변경했습니다. Reverse Proxy로 Nginx도 검토했지만, Ceph MGR의 Active-Standby 특성과 Health Check 기반 라우팅을 고려해 HAProxy + Keepalived 조합을 선택했습니다.

**HAProxy가 담당하도록 분리한 책임**
- Active MGR 선택
- Health Check 기반 Standby / Down MGR 제외
- Backend에 단일 진입점(VIP) 제공

**Backend에서 제거한 로직**
- 303 Redirect 파싱
- Host 순회 Fallback
- 재시도 기반 MGR 재호출
- Active MGR 갱신 이벤트

정리하면, 가용성 판단과 MGR 라우팅은 인프라 책임으로 분리하고, Backend는 인증과 API 호출 정책에 집중하도록 구조를 변경했습니다. 이를 통해 Backend 코드 복잡도를 줄이고, 장애 원인을 인프라와 애플리케이션 레이어로 구분할 수 있는 구조를 만들었습니다.

---

## 3단계: 애플리케이션 회복력 강화 (Resilience4j)

HAProxy를 도입하면서 Active MGR 선택과 장애 MGR 제외는 인프라 책임으로 분리했습니다. Backend는 더 이상 “어느 MGR로 요청을 보낼지” 판단하지 않고, VIP만 호출하면 되는 구조가 되었습니다.

하지만 여전히 애플리케이션 레벨에서 해결해야 할 문제가 남아 있었습니다. 외부 시스템 호출의 실패 가능성을 전제로, Backend는 어떤 실패를 재시도하고, 어떤 실패를 즉시 차단하며, 어떤 장애를 클러스터 단위로 격리할지에 대한 정책을 가져야 했습니다.

이를 위해 Resilience4j의 Retry와 CircuitBreaker를 도입했습니다. Spring Retry는 Retry 중심의 기능에 가깝고, Hystrix는 신규 도입에 적합하지 않다고 판단했습니다. 반면 Resilience4j는 Retry, CircuitBreaker, Metrics 연동을 모듈 단위로 구성할 수 있어 현재 요구사항에 적합했습니다.

현재 장애 패턴에서는 Bulkhead, RateLimiter, TimeLimiter보다 일시 오류 재시도와 장애 클러스터 차단이 더 중요하다고 판단해 Retry + CircuitBreaker 조합만 적용했습니다.

실행 순서는 `CB(Retry(Supplier))`로 구성했습니다. Retry가 먼저 502/503 같은 일시 오류를 복구 시도하고, 최종 실패 결과만 CircuitBreaker의 실패율에 반영되도록 하기 위해서입니다. 반대로 CircuitBreaker를 Retry 내부에 두면 각 재시도 실패가 모두 CircuitBreaker에 기록되어, 장애가 과도하게 빠르게 OPEN될 수 있습니다.

```yaml
resilience4j:
  retry:
    retryExceptions:
      - BadGateway          # 502: HAProxy 일시 오류 (재시도 가능)
      - ServiceUnavailable  # 503: MGR 준비 안 됨 (재시도 가능)
    ignoreExceptions:
      - InternalServerError # 500: 재시도 효용 낮음
  circuitbreaker:
    ignoreExceptions:
      - InvalidCredentialsException  # 인증 오류 ≠ 시스템 장애
```

또한 CircuitBreaker는 Provider별로 분리했습니다. 하나의 Ceph 클러스터에 장애가 발생하더라도 해당 Provider의 CircuitBreaker만 OPEN되고, 다른 클러스터 요청은 정상적으로 처리되도록 하기 위해서입니다.

```java
String key = "ceph-" + providerUuid;  // 클러스터마다 독립 CB
Supplier<T> decorated = Retry.decorateSupplier(retry, supplier);
decorated = CircuitBreaker.decorateSupplier(circuitBreaker, decorated);
return decorated.get();
```

설정값이 실제 장애 상황에서 의도대로 동작하는지도 테스트로 검증했습니다. 예를 들어 인증 오류가 CircuitBreaker 실패율에 포함되지 않는지, 502/503은 재시도되는지, 최소 호출 수 이전에는 CircuitBreaker가 OPEN되지 않는지 등을 장애 시나리오 기반 테스트로 확인했습니다.

![개선 구조 시퀀스 다이어그램](/assets/img/ceph-seq-after.png)
_개선 구조 — Backend는 VIP만 호출하고, 실패 처리는 Resilience4j의 `CB(Retry(Supplier))` 정책으로 다룹니다. 401은 토큰 재발급, 502/503은 Retry, 반복 실패로 CircuitBreaker가 OPEN되면 Fast-Fail로 처리합니다._

---

## 4단계: 서비스 관점의 장애 감지 (커스텀 메트릭)

HAProxy와 Resilience4j를 통해 Ceph 장애가 Backend 요청 흐름에 직접 전파되지 않도록 구조를 개선했습니다. 하지만 운영 관점에서는 여전히 중요한 문제가 남아 있었습니다. 장애를 차단하더라도, 어떤 Provider에서 어떤 이유로 장애가 발생하고 있는지 빠르게 감지할 수 있어야 했습니다.

![전체 아키텍처](/assets/img/ceph-architecture.png)
_전체 아키텍처 — HAProxy + Keepalived(VIP) 단일 진입점부터 Prometheus·Alertmanager·Slack 기반 관찰 가능성 파이프라인까지._

CPU, 메모리, 헬스체크 같은 기본 지표만으로는 특정 Provider의 CircuitBreaker가 OPEN 상태로 유지되거나, 특정 클러스터 호출 실패율이 증가하는 상황을 충분히 감지하기 어려웠습니다. 서버 자체는 정상 동작하더라도, 서비스 관점에서는 이미 특정 클러스터 요청이 실패하고 있을 수 있기 때문입니다.

그래서 이 서비스에서 관찰해야 할 장애를 직접 정의하고, Ceph API 호출 경계에서 커스텀 메트릭으로 계측하기로 했습니다.

### 단일 측정 경계 설정

모든 Ceph Dashboard API 호출은 `CephClient.call()`을 통과합니다. 따라서 메트릭도 이 단일 경계에서만 기록했습니다. 호출부마다 메트릭을 분산해서 기록하면 누락과 중복이 발생할 수 있고, API 호출 로직과 관측 로직이 여러 곳에 섞일 수 있기 때문입니다.

```java
public <T> CephCallResult<T> call(CephRequest<T> request) {
    String component = CephApiNormalizer.componentLabel(request);
    String provider = CephApiNormalizer.providerLabel(request.getProvider());

    long start = System.nanoTime();
    try {
        CephCallResult<T> result = ...;
        cephMetrics.recordSuccess(component, provider, System.nanoTime() - start);
        return result;
    } catch (Exception e) {
        cephMetrics.recordFailure(component, provider, e, System.nanoTime() - start);
        throw handleException(request.getProvider(), request.getPath(), e);
    }
}
```

기록하는 메트릭은 세 가지입니다.
- `ceph_api_requests_total{component, provider, result}` — API 요청 수
- `ceph_api_failures_total{component, provider, reason}` — 실패 **원인별** 요청 수
- `ceph_api_duration_seconds{component, provider}` — 응답 지연 시간(bucket: 100ms/300ms/1s/2s/3s/5s)

핵심은 실패를 단순히 집계하는 것이 아니라, `reason` 라벨로 원인을 분류하고, `component` 라벨로 장애가 발생한 Ceph 영역을 구분한 점입니다.

```java
String classifyReason(Throwable error) {
    if (error instanceof CallNotPermittedException) return "circuit_open";
    if (hasCause(error, HttpClientErrorException.class)) return "http_client";
    if (hasCause(error, HttpServerErrorException.class)) return "http_server";
    if (hasCause(error, SocketTimeoutException.class))  return "timeout";
    if (hasCause(error, ResourceAccessException.class)) return "network";
    return "unknown";
}
```

### 라벨 카디널리티 통제

커스텀 메트릭을 추가할 때는 라벨 카디널리티도 함께 통제해야 했습니다. `component` 라벨에 raw path를 그대로 사용하면 `/api/block/image/{id}`처럼 값이 계속 늘어나 Prometheus 저장 비용과 조회 비용이 증가할 수 있습니다.

이를 방지하기 위해 API path를 Ceph component 단위로 정규화했습니다. 예를 들어 `/api/health`, `/api/host`, `/api/osd`, `/api/block`, `/api/rgw` 같은 요청은 각각 `health`, `host`, `osd`, `block`, `rgw`로 수렴시키고, 사전에 정의하지 않은 path는 `other`로 분류했습니다.

그 결과 메트릭은 component, provider, result/reason 기준으로 조회할 수 있게 되었습니다.

```text
ceph_api_requests_total{component="health", provider="cluster-a", result="failure"} 
ceph_api_requests_total{component="osd", provider="cluster-a", result="success"} 
ceph_api_failures_total{component="block", provider="cluster-a", reason="timeout"}
```

이를 통해 특정 Provider에서 장애가 발생했을 때, 단순히 “Ceph 호출이 실패했다”가 아니라 어느 클러스터의 어떤 Ceph component에서 어떤 원인으로 실패했는지까지 확인할 수 있게 했습니다.

### 감지 파이프라인 구성

Backend에서 노출한 메트릭은 Prometheus가 scrape하고, Alert Rule을 통해 장애 조건을 평가하도록 구성했습니다. 이후 Alertmanager와 Slack Webhook을 연결해 특정 Provider의 CircuitBreaker OPEN, 실패율 급증, fast-fail 증가를 알림으로 받을 수 있도록 했습니다. Grafana에서는 Provider별 호출 수, 실패율, 지연 시간 추세를 확인할 수 있도록 대시보드를 구성했습니다.

결과적으로 장애를 수동 확인에 의존하던 구조에서, Provider별 장애 상태를 메트릭과 알림으로 감지하는 구조로 개선했습니다. 또한 component, provider, reason 라벨을 통해 어느 클러스터의 어떤 Ceph component에서 어떤 원인으로 실패했는지 1차 분류할 수 있게 했습니다.

---

## 결과

장애 상황 기준으로 k6 부하 테스트를 다시 수행해 비교했습니다.

| 항목 | 개선 전 | 개선 후 | 의미 |
|------|--------|--------|------|
| 평균 응답 시간 | 5.04s | 36.5ms | 장애 요청 즉시 차단 |
| P95 레이턴시 | 10.03s | 78ms | tail latency 감소 |
| dropped_iterations | 989 | 11 | 요청 backlog 해소 |
| 최대 VUs | 50 (max 도달) | 16 | 요청 처리 지연과 리소스 점유 감소 |
| 장애 클러스터 fast-fail | 불가 | 98~99% (100ms 이내) | 장애 즉시 실패 반환 |
| 장애 감지 | 수동 | 메트릭 + 알림 자동 감지 | 모니터링 확보 |

이 수치는 정상 상황의 성능 개선이 아니라, **장애 상황에서의 회복 탄력성 개선**을 의미합니다.

- **서비스 가용성** — 특정 클러스터 장애가 전체 서비스로 전파되지 않도록 제한했습니다.
- **운영 효율** — 장애 원인을 인프라, Ceph 응답, 애플리케이션 정책 관점에서 구분할 수 있게 했습니다.
- **코드 복잡도** — Backend 내부의 Redirect/Fallback/Active MGR 갱신 로직 약 200줄을 제거했습니다.
- **확장성** — 신규 클러스터 추가 시 Provider별 CircuitBreaker가 독립적으로 적용되어 장애 격리 구조를 유지할 수 있습니다.

---

## 회고

처음에는 이 문제를 `redirect`, `retry`, `fallback` 로직을 어떻게 더 안정적으로 개선할지의 문제로 봤습니다. 하지만 장애 상황을 재현하고 구조를 분석하면서, 핵심은 개별 예외 처리 코드가 아니라 Active MGR 선택과 failover 처리 책임이 Backend 안에 들어와 있었다는 점이라는 것을 알게 됐습니다.

이번 경험을 통해 장애 대응은 예외 처리 코드를 늘리는 방식보다, 장애가 전파되지 않도록 레이어별 책임을 분리하고, 장애 클러스터를 격리하며, 그 상태를 측정 가능하게 만드는 것이 중요하다는 점을 배웠습니다.

또한 Resilience4j 설정과 커스텀 메트릭을 테스트와 대시보드로 검증하면서, 복잡한 설정 역시 코드처럼 동작을 증명해야 신뢰할 수 있다는 것을 체감했습니다. 이번 개선을 통해 안정적인 시스템을 만들기 위해서는 장애 처리 코드를 늘리는 것보다, **장애가 Backend 요청 흐름으로 전파되지 않도록 책임을 분리하고, 장애 범위를 제한하며, 장애 상태를 빠르게 감지할 수 있는 구조**가 필요하다는 점을 배웠습니다.