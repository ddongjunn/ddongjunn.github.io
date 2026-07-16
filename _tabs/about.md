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

백엔드 5년차로, Java/Spring Boot 기반 Ceph 분산 스토리지 플랫폼을 개발·운영하고 있습니다. 장애 상황에서도 예측 가능하게 동작하는 서버 구조를 중요하게 생각합니다. 외부 시스템 연동·동시성 충돌·장애 전파 문제를 다루며, 기능의 정상 동작뿐 아니라 실패 시 시스템의 반응까지 함께 고려해왔습니다. 트랜잭션 정합성, 책임 레이어 분리, 장애 격리처럼 운영 환경에서 드러나는 문제를 구조 관점에서 해결하는 데 관심이 많습니다.

## Skills

| Category | Stack |
|---|---|
| Backend | Java, Spring Boot, JPA, MyBatis, Resilience4j |
| Database | MySQL, Redis |
| Infra / Ops | Docker, Ansible, Jenkins, Ceph, HAProxy, Prometheus, Grafana, Alertmanager, k6 |

## Experience

### 오케스트로 주식회사 · 분산스토리지팀

_Backend Engineer · 정규직 · 2024.09 ~ 재직 중_

Ceph 기반 분산 스토리지 플랫폼의 백엔드 서비스를 개발·운영하며, 장애 회복 탄력성과 운영 효율성을 개선하고 있습니다.

- Ceph 클러스터 장애 전파 차단 아키텍처 재설계로 장애 상황 **P95 레이턴시 10.03s → 78ms** 개선
- Ceph Exporter, Node Exporter, Micrometer 기반 커스텀 메트릭 수집 및 Slack 장애 알림 구성
- go-ceph Custom Container 기반 확장 API로 이미지 위치 분석 **29.4s → 0.9s** 개선
- Ansible 기반 Ceph 클러스터 배포 자동화로 구축 시간 **반나절 → 5~10분** 단축

### 주식회사 코비젼 · 기술지원팀

_Backend Engineer · 정규직 · 2021.10 ~ 2024.08_

B2B 그룹웨어 솔루션의 백엔드 개발 및 유지보수를 담당하며, 성능 병목 분석, ERP 연동 정합성 개선, 운영 자동화를 수행했습니다.

- Scouter APM 기반 병목 분석과 인덱스 설계로 API 응답 시간 **3.375s → 0.016s** 개선
- AOP 기반 ERP 공통 인증 모듈로 API별 인증 중복 제거 및 결재 정합성 개선
- 트랜잭션 커밋 이후 비동기 이벤트로 메일 발송을 계정 발급 흐름에서 분리해 외부 시스템 실패 격리
- MyBatis 환경에서 version 기반 낙관적 락을 직접 구현해 동시 예약 정합성 보장

## Projects
{: .page-break}

### Ceph 클러스터 장애 전파를 차단한 서버 고가용성 구조 개선
{: .project-title}

_오케스트로 · 2025.10 ~ 2026.03 · Java, Spring Boot, Resilience4j, HAProxy, Prometheus, k6_

![Ceph 고가용성 구조와 모니터링 파이프라인 다이어그램](/assets/img/ceph-architecture.png)
_HAProxy/VIP, Resilience4j, Prometheus 기반으로 장애 전파를 차단한 개선 구조_

#### 문제

Ceph Active-Standby MGR 선택, 303 Redirect, retry, host fallback을 백엔드에서 직접 처리하면서 클러스터 장애가 전체 서비스 응답 지연으로 전파되는 문제가 있었습니다.

#### 분석

k6 장애 시나리오 테스트로 기존 redirect/fallback 구조가 장애 상황에서 응답 지연을 증폭시키는 원인임을 검증하고, VIP 단일 진입점 구조와 비교해 핵심 문제가 재시도 로직이 아닌 Active MGR 선택·failover 책임 배치에 있음을 확인했습니다.

#### 해결

- Single-flight 패턴을 적용해 동일 Provider 대상 토큰 중복 발급과 DB Optimistic Lock 충돌 완화
- HAProxy + Keepalived(VIP)를 도입해 MGR 선택·failover 책임을 인프라 레이어로 이관
- Resilience4j Retry + CircuitBreaker를 Provider별로 독립 구성해 특정 클러스터 장애의 전파 차단 및 fast-fail 구조 확보
- component, provider, reason 기준 커스텀 메트릭을 정의하고 Prometheus·Alertmanager 기반 장애 감지 파이프라인 구축

#### 성과

- 장애 상황 기준 P95 레이턴시 **10.03s → 78ms** 단축
- 특정 Ceph 클러스터 장애가 다른 Provider 요청으로 전파되지 않는 장애 격리 구조 확보
- Provider별 CircuitBreaker OPEN, 실패율 증가, fast-fail 증가를 메트릭과 알림으로 감지하는 운영 구조 확보

#### 참고

[Ceph 클러스터 장애 전파를 차단한 서버 고가용성 구조 개선](https://ddongjunn.github.io/posts/cluster-failure-isolation/)

### Ceph Custom Container 기반 확장 API 배포 구조 설계
{: .project-title}

_오케스트로 · 2025.05 ~ 2025.07 · go, go-ceph, Ceph, RADOS, Docker_

![go-ceph Custom Container 배포 구조 다이어그램](/assets/img/go-ceph-architecture.png)
_Developer 이미지 빌드·푸시 → cephadm 오케스트레이션으로 Custom Container 배포, Prometheus 메트릭 수집_

#### 문제

Ceph MGR API 중심의 연동 구조에서는 RBD 이미지 수 증가 시 리스트 조회 시간이 3.2s까지 증가했고, 데이터 저장 위치 분석처럼 MGR API가 제공하지 않는 기능을 서비스 레벨에서 구현하기 어려웠습니다.

#### 분석

MGR Module은 Ceph 내부 플러그인 방식이라 Python 전용 제약과 복잡한 배포·갱신 절차로 운영 부담이 컸고, librados 직접 접근과 느슨한 클러스터 결합이 가능한 독립 컨테이너 방식이 더 적합하다고 판단했습니다.

#### 해결

- go-ceph 기반 API 서버를 구현하고 Ceph Custom Container 형태로 클러스터 내부에 배포
- RBD 이미지의 실제 사용 객체만 대상으로 분석 범위를 축소해 불필요한 I/O 제거
- 객체 저장 위치 분석 단계를 워커 풀 기반 병렬 처리로 분산 수행

#### 성과

- RBD 이미지 리스트 조회 **3.2s → 1.2s** 단축
- 11.8 GiB 이미지(약 128,000 객체) 위치 분석 **29.4s → 0.9s** 단축
- MGR API에서 제공하지 않는 클러스터 분석 기능을 독립 API 서버로 확장 가능한 구조 확보

#### 참고

[github.com/ddongjunn/go-ceph](https://github.com/ddongjunn/go-ceph)

### Ansible 기반 Ceph 클러스터 배포 자동화
{: .project-title}

_오케스트로 · 2025.03 ~ 2025.04 · Ansible, Ceph, Shell, Jinja2_

#### 문제

설정 변경이 잦은 초기 개발 단계에서 Ceph 클러스터를 반복 생성·폐기해야 했지만 설치부터 노드별 설정·서비스 구성까지 수작업으로 진행되어 클러스터 하나 구축하는 데 반나절 이상 소요됐고 환경 편차가 발생했습니다.

#### 분석

반복적인 클러스터 구성 변경에 대응하려면 단순 설치 자동화가 아니라, 클러스터 설정·서비스 배포·리소스 구성을 코드로 표준화하고 환경별 옵션을 변수·템플릿으로 분리하는 Ansible 구조가 필요하다고 판단했습니다.

#### 해결

- Ansible Role 기반으로 bootstrap, 공통 설정, 서비스 배포, 상태 검증 플로우 분리
- group_vars/all.yml 전역 변수로 노드 구성·네트워크·서비스 옵션을 선언적으로 관리
- Jinja2 템플릿으로 서비스 spec을 동적 생성해 환경별 구성 변경 대응

#### 성과

- 반나절 이상 소요되던 Ceph 클러스터 구축 시간을 **3노드 기준 5~10분**으로 단축
- 환경 편차 없이 동일한 구성의 클러스터를 반복 생성 가능한 구조 확보

#### 참고

[github.com/ddongjunn/ceph-ansible](https://github.com/ddongjunn/ceph-ansible)

### ERP 연동 공통 인증 모듈 및 전자결재 정합성 개선
{: .project-title}

_코비젼 · 2023.11 ~ 2024.05 · Java, Spring, AOP_

#### 문제

전자결재 상신 시 외부 ERP API별 인증 로직이 중복 구현되어 있었고, ERP 호출 실패와 무관하게 결재가 승인되어 데이터 정합성이 깨질 수 있는 구조였습니다.

#### 분석

인증 로직은 API별 개별 구현보다 공통 모듈로 분리하는 것이 유지보수·확장성 측면에서 적합하다고 보고, ERP 응답을 기준으로 결재 승인 흐름을 제어해 기존 결재 프로세스의 변경 범위를 최소화하는 방향으로 접근했습니다.

#### 해결

- AOP 기반 공통 인증 모듈로 ERP API 로그인·인증 체크 로직 일원화
- ERP 응답 코드 기준 예외 처리 규칙 정의 및 결재 승인 흐름 제어
- ERP API 요청·응답 로깅 추가로 장애 추적성 개선

#### 성과

- 신규 ERP API 추가 시 공통 인증 모듈 재사용 가능 구조 확보
- ERP 호출 실패 시 후속 결재 진행을 차단해 결재-ERP 간 데이터 정합성 확보

### APM 도입 및 쿼리 튜닝을 통한 성능 최적화
{: .project-title}

_코비젼 · 2023.05 ~ 2023.08 · Java, Spring, MySQL, Scouter APM_

#### 문제

SaaS 그룹웨어 환경에서 다수 고객사의 전자결재 데이터가 누적되며, 특정 전자결재 설정 API에서 응답 지연이 발생했습니다.

#### 분석

Scouter APM으로 API 구간별·SQL 단위 수행 시간을 측정하고, 실행계획(EXPLAIN)을 비교해 Full Table Scan, Using Temporary, Using Filesort가 발생하는 병목 조인 구간을 식별했습니다.

#### 해결

- 병목 조인 구간을 기준으로 다중 조인 쿼리 구조 재구성
- 카디널리티와 조회 조건을 기준으로 인덱스 재설계
- 실행계획을 Index Range Scan 중심으로 개선

#### 성과

- 해당 API 응답 시간 **3.375s → 0.016s** 개선
- Full Table Scan 기반 조회를 Index Range Scan 중심 구조로 개선

## Open Source

- **Spring Boot 오픈소스 기여** · PR [#48347](https://github.com/spring-projects/spring-boot/pull/48347) (4.1.0-M1 반영)<br>
`MustacheResourceTemplateLoader` 템플릿 인코딩 처리 개선 및 public API 호환성 보완 후 머지

## Education

- 학점은행제 · 정보통신공학 학사 (2022.10 ~ 2023.02)
- 동양미래대학 · 정보통신공학과 전문학사 (2012.03 ~ 2021.02)
