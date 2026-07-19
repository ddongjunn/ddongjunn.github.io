---
title: About
icon: fas fa-info-circle
layout: resume
permalink: /about/
order: 4
name: 이동준
role: Backend Engineer
photo: /assets/img/avatar.png
contacts:
  - icon: "✉️"
    label: ddongjunn@gmail.com
    url: mailto:ddongjunn@gmail.com
  - icon: "💻"
    label: github.com/ddongjunn
    url: https://github.com/ddongjunn
  - icon: "📝"
    label: jhost.tistory.com
    url: https://jhost.tistory.com/
---

## Summary

장애의 영향이 시스템 전체로 확산되지 않도록 구조를 설계하는 5년 차 백엔드 엔지니어입니다. Java/Spring 기반 B2B 솔루션과 Ceph 기반 분산 스토리지 플랫폼을 개발하고 운영하며 데이터 정합성을 확보하고, 동시성 충돌과 장애 전파 문제를 해결해왔습니다. 기능의 정상 동작뿐 아니라 문제가 발생했을 때의 시스템 반응과 영향 범위까지 고려해 예측 가능하게 동작하는 서비스를 만들어왔습니다.

## Skills

| Category | Stack |
|---|---|
| Backend | Java, Spring Boot, JPA, MyBatis, Resilience4j |
| Database | MySQL, Redis |
| Infra / Ops | Docker, Ansible, Jenkins, Ceph, HAProxy, Prometheus, Grafana, Alertmanager |

## Experience

### 오케스트로 
_Backend Engineer · 분산스토리지팀 · 2024.09 ~ 재직 중_

Ceph 기반 분산 스토리지 플랫폼의 백엔드 서비스를 개발하고 운영하며, 클러스터 장애 격리와 분산 환경의 데이터 일관성 확보, 성능 개선을 수행하고 있습니다.

- 클러스터별 CircuitBreaker로 장애 영향을 격리해 장애 상황 P95 레이턴시를 **10.03s에서 78ms로 단축**
- Java 21 가상 스레드 Pinning 원인을 MySQL Connector/J의 `synchronized` 구간으로 확인하고 드라이버 업그레이드로 해결
- 로컬 메모리 기반 클러스터 정보 캐시를 Redis로 전환해 **멀티 인스턴스 간 데이터 일관성 확보**
- Ansible로 Ceph 클러스터 구축과 삭제를 자동화해 동일한 구성으로 반복 재구축할 수 있도록 작업 절차 표준화
- go-ceph 기반 스토리지 분석 API를 개발해 이미지 저장 위치 분석 시간을 **29.4s에서 0.9s로 단축**

### 코비젼
_Backend Engineer · 기술지원팀 · 2021.10 ~ 2024.08_

B2B 그룹웨어 솔루션을 개발·유지보수하며 ERP 연동, 성능 병목 개선, 업무 자동화, 외부 시스템 실패 격리와 동시성 제어를 수행했습니다.

- Scouter APM과 실행계획 분석으로 병목 쿼리와 인덱스를 개선해 API 응답 시간을 **3.375s에서 16ms로 단축**
- 고객사별 제품 설정을 Shell 스크립트로 자동화해 반나절 이상 걸리던 서버 설정 작업을 **5분으로 단축**
- 메일 발송을 커밋 이후 비동기 이벤트로 분리해 부가 로직 실패로 **핵심 발급 트랜잭션이 롤백되는 문제 해결**
- MyBatis 환경에서 version 조건 기반 낙관적 락을 직접 구현해 동시 장비 예약 충돌 감지

## Projects
{: .page-break}

### Ceph 클러스터 장애 전파를 차단한 서버 고가용성 구조 개선
{: .project-title}
_오케스트로 · 2025.10 ~ 2026.03 · Java 21, Spring Boot 3.5, Resilience4j, HAProxy, Prometheus_

![Ceph 고가용성 구조와 모니터링 파이프라인 다이어그램](/assets/img/ceph-architecture.png)
_HAProxy/VIP, Resilience4j, Prometheus 기반 장애 격리 및 감지 구조_

#### 문제

Ceph Active-Standby MGR 선택, 303 Redirect, retry, host fallback을 백엔드가 직접 처리하면서 특정 클러스터 장애로 인한 요청 지연이 전체 서비스로 전파됐습니다.

#### 분석

동일한 Ceph 장애 상황에서 기존 redirect, retry, host 순회 구조는 단건 응답에 1,150ms가 걸렸지만, 해당 로직을 제거한 단순 호출은 7.2ms였습니다. 이를 통해 기존 복합 처리 구조가 장애 상황의 지연을 증폭하고 있음을 확인하고, Active MGR 선택과 failover 책임이 백엔드 요청 흐름에 포함된 구조를 핵심 원인으로 판단했습니다.

#### 해결

- Single-flight 패턴을 적용해 동일 클러스터 대상 토큰 중복 발급과 DB Optimistic Lock 충돌 완화
- **HAProxy와 Keepalived 기반 VIP 단일 진입점**을 구성해 Active MGR 선택과 failover 책임을 인프라 레이어로 이관
- **클러스터별로 Retry와 CircuitBreaker**를 구성해 일시적 오류는 재시도하고, 장애가 지속되면 해당 클러스터 요청을 fast-fail 처리
- `provider`, `component`, `reason` 기준 커스텀 메트릭을 정의해 장애 감지 파이프라인 구축

#### 성과

- 장애 상황 P95 레이턴시를 **10.03s → 78ms**로 단축하고, 장애 클러스터 요청의 **98~99%를 100ms 이내 fast-fail** 처리
- 특정 Ceph 클러스터 장애가 다른 클러스터 요청으로 전파되지 않는 장애 격리 구조 확보
- 클러스터별 CircuitBreaker OPEN, 실패율 증가, fast-fail 증가를 메트릭과 알림으로 감지하는 운영 구조 확보

#### 참고

[Ceph 클러스터 장애 전파를 차단한 서버 고가용성 구조 개선](https://ddongjunn.github.io/posts/cluster-failure-isolation/)

### Ceph 스토리지 분석 API 개발 및 성능 개선
{: .project-title}
_오케스트로 · 2025.05 ~ 2025.07 · Go, go-ceph, Ceph, RADOS, Docker_

![스토리지 분석 API 배포 구조](/assets/img/go-ceph-architecture.png)
_별도 API 서버 구현부터 cephadm 기반 컨테이너 배포까지 연결한 구조_

#### 문제

기존 Ceph API는 블록 스토리지 이미지(RBD)의 데이터 저장 위치를 분석하는 기능을 제공하지 않았습니다. 이를 위해 Ceph 클러스터에 직접 접근해 객체별 저장 위치를 조회하는 기능을 구현했지만, 초기 로직은 전체 객체를 순차 조회해 29.4s가 걸렸습니다.

#### 분석

Ceph MGR Module은 개발 언어와 배포에 제약이 있어 해당 기능을 별도 API 서버로 구현하는 방향을 선택했습니다. 초기 분석 로직의 병목을 줄이기 위해 실제 사용 데이터만 선별하고, 서로 독립적인 조회 작업을 병렬화하도록 설계했습니다.

#### 해결

- go-ceph 기반 API 서버를 구현해 기존 Ceph API에 없는 저장 위치 분석 기능 확장
- `DiffIterate`로 실제 데이터가 할당된 객체만 선별해 불필요한 조회 제거
- 객체별 저장 위치 조회 단계를 워커 풀로 병렬 처리
- API 서버를 컨테이너 이미지로 구성하고 cephadm Custom Container로 클러스터 내부에 배포

#### 성과

- 블록 스토리지 이미지의 데이터 저장 위치 분석 시간을 **29.4s에서 0.9s로 단축**
- 기존 Ceph API와 분리된 기능 확장 및 배포 구조 확보

#### 참고

[github.com/ddongjunn/go-ceph](https://github.com/ddongjunn/go-ceph)

### Ansible 기반 Ceph 클러스터 생성 및 삭제 자동화
{: .project-title}
_오케스트로 · 2025.03 ~ 2025.04 · Ansible, Ceph, Shell, Jinja2_

#### 문제

설정 변경이 잦은 초기 개발 단계에서 Ceph 클러스터를 반복해서 폐기하고 재구축해야 했습니다. 호스트 준비부터 서비스 구성까지 수작업으로 진행해 한 번에 반나절 이상이 걸렸고, 작업 순서 누락과 노드별 설정 편차가 발생했습니다.

#### 분석

여러 노드의 작업 순서와 호스트 및 서비스 설정을 일관되게 관리하고, 배포 후 상태 확인과 삭제까지 하나의 흐름으로 자동화하기 위해 Ansible을 선택했습니다.

#### 해결

- 공통 설정, bootstrap, 서비스 배포, 상태 확인을 Ansible Role로 분리
- 노드, 네트워크, 서비스 설정을 `group_vars`와 Jinja2 템플릿으로 관리
- 별도 실행 스크립트에서 클러스터 배포와 삭제를 선택해 수행할 수 있도록 구성

#### 성과

- Ceph 클러스터 구축 시간을 **반나절 이상에서 5~10분으로 단축**
- 설정 변경 후 같은 절차로 클러스터를 폐기하고 재구축할 수 있도록 작업 과정 표준화

#### 참고

[github.com/ddongjunn/ceph-ansible](https://github.com/ddongjunn/ceph-ansible)

### Scouter APM 기반 병목 분석 및 쿼리 튜닝
{: .project-title}
_코비젼 · 2023.05 ~ 2023.08 · Java 8, Spring 4.x, MyBatis, MySQL, Scouter APM_

#### 문제

다수 고객사의 전자결재 데이터가 누적되면서 특정 전자결재 설정 API의 응답 시간이 3.375s까지 증가했습니다.

#### 분석

Scouter APM으로 API 구간별 실행 시간을 측정해 병목이 다중 조인 SQL에 있음을 확인했습니다. 실행계획을 분석한 결과 `Full Table Scan`, `Using Temporary`, `Using Filesort`가 발생하고 있었습니다.

#### 해결

- 병목 조인 구간을 중심으로 다중 조인 쿼리와 조회 조건 재구성
- 데이터 카디널리티와 조회 조건을 기준으로 **인덱스 재설계**
- 변경 전후 실행계획을 비교해 `Index Range Scan` 적용 확인

#### 성과

- 해당 API 응답 시간을 **3.375s에서 16ms로 단축**

## Open Source

- Spring Boot · PR [#48347](https://github.com/spring-projects/spring-boot/pull/48347) (4.1.0-M1 반영)<br>
`MustacheResourceTemplateLoader` 템플릿 인코딩 처리 개선 및 public API 호환성 보완 후 머지

## Education

- 학점은행제 · 정보통신공학 학사 (2022.10 ~ 2023.02)
- 동양미래대학 · 정보통신공학과 전문학사 (2012.03 ~ 2021.02)
