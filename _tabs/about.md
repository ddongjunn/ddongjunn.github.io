---
title: About
icon: fas fa-info-circle
order: 4
---

## Backend Engineer

**백엔드 5년차**로, Java/Spring Boot 기반 **Ceph 분산 스토리지 플랫폼을 개발·운영**하고 있습니다. 장애 상황에서도 예측 가능하게 동작하는 서버 구조를 중요하게 생각하며, 외부 시스템 연동·동시성 충돌·장애 전파 문제를 경험하며 기능이 정상 동작하는 것뿐 아니라 실패했을 때 시스템이 어떻게 반응해야 하는지도 함께 고민해왔습니다. 트랜잭션 정합성, 책임 레이어 분리, 장애 격리처럼 운영 환경에서 드러나는 문제를 구조 관점에서 해결하는 데 관심이 많습니다.

<i class="fa-solid fa-envelope"></i> [ddongjunn@gmail.com](mailto:ddongjunn@gmail.com) &nbsp;·&nbsp; <i class="fa-brands fa-github"></i> [github.com/ddongjunn](https://github.com/ddongjunn) &nbsp;·&nbsp; <i class="fa-solid fa-pen-nib"></i> [jhost.tistory.com](https://jhost.tistory.com)

---

## 기술 스택

`Java` `Spring Boot` `JPA` `MySQL` `Redis` `Prometheus` `Docker` `Ansible`

---

## 경력

### 오케스트로 주식회사 · 분산스토리지팀

`사원 · 정규직 · 2024.09 ~ 재직 중 (약 1년 9개월)`

Ceph 기반 분산 스토리지 플랫폼의 백엔드 서비스를 개발·운영하며, 장애 회복 탄력성과 운영 효율성을 개선하고 있습니다.

- Ceph 클러스터 장애 전파 차단 아키텍처 재설계로 **P95 레이턴시 10.03s → 78ms**
- Ceph Exporter·Node Exporter 연동 및 Micrometer 기반 커스텀 메트릭 수집 구조 구축
- Ceph 운영 지표 대시보드 및 Alertmanager 기반 장애 알림 기능 개발
- Ceph Custom Container 기반 확장 API 설계 (**이미지 분석 29.4s → 0.9s**)
- Ansible 기반 Ceph 클러스터 배포 자동화

### 주식회사 코비젼 · 기술지원팀

`사원 · 정규직 · 2021.10 ~ 2024.08 (2년 11개월)`

Java, Spring 기반 B2B 솔루션의 백엔드 개발 및 유지보수를 담당하며 제품 품질과 운영 효율성을 개선했습니다.

- 슬로우 쿼리 분석 및 인덱스 설계 (**API 응답시간 3.375s → 0.016s**)
- Scouter APM 도입으로 병목 구간 분석 체계 구축
- ERP 연동 시스템 개발 및 데이터 정합성 문제 해결
- 반복 작업 자동화로 운영 효율성 향상

---

## 프로젝트

### Ceph 클러스터 장애 전파를 차단한 서버 고가용성 구조 개선

`오케스트로 · 2025.10 ~ 2026.03`

장애 처리 책임을 **인프라(HAProxy+VIP)·애플리케이션(Resilience4j)으로 재배치**하고, 커스텀 메트릭 기반 감지 체계를 구축해 장애 상황 **P95 레이턴시 10.03s → 78ms** 단축 및 클러스터 간 장애 전파를 차단했습니다.

**관련 글** · [Ceph 클러스터 장애 전파를 차단한 서버 고가용성 구조 개선](/posts/cluster-failure-isolation/)

<details open markdown="1">
<summary>자세히 보기</summary>

**Problem**

Ceph Active-Standby의 MGR 선택, redirect, fallback을 백엔드에서 직접 처리하면서 클러스터 장애 시 `redirect → retry → host fallback` 로직이 반복 실행됐고, 특정 클러스터 장애가 서비스 전체 응답 지연으로 이어졌습니다.

**Analyze**

k6 기반 장애 시나리오 테스트 과정에서 토큰 중복 발급과 DB Optimistic Lock 충돌로 인해 부하 테스트 자체가 신뢰하기 어려운 상태임을 확인했습니다. Single-flight 패턴으로 동시성 문제를 제거한 뒤 재측정한 결과, 기존 redirect/fallback 구조와 VIP 기반 단일 진입점 호출 구조 간 응답시간 차이(1,150ms → 7.2ms)를 확인했고, 문제의 본질이 재시도 로직이 아니라 책임 레이어 배치에 있음을 확인했습니다.

**Action**

- Single-flight 패턴으로 토큰 발급 중복 제거 (현재 인스턴스 규모와 stateless 토큰 구조를 고려했을 때, Redis 분산 락보다 인스턴스 내부 중복 제거가 비용 대비 효율적이라고 판단)
- HAProxy + Keepalived(VIP)로 MGR 선택·failover를 인프라 레이어로 이관하고, Health Check 기반 단일 진입점 구조 구성
- Resilience4j Retry + CircuitBreaker 도입, Provider별 독립 CB로 클러스터 간 장애 격리 및 fast-fail 구조 확보
- `component`, `provider`, `reason` 기준 커스텀 메트릭을 정의하고 Prometheus, Alertmanager, Slack 기반 장애 감지 파이프라인 구성

**Result**

- 장애 상황 기준 P95 레이턴시 **10.03s → 78ms** 단축
- 특정 클러스터 장애가 다른 클러스터에 **전파되지 않는 구조** 확보
- Provider별 CircuitBreaker OPEN, 실패율 증가, fast-fail 증가를 **메트릭과 Slack 알림으로 감지하는 운영 구조** 확보

</details>

### Ceph Custom Container 기반 확장 API 배포 구조 설계

`오케스트로 · 2025.05 ~ 2025.07`

librados 직접 접근이 가능한 go-ceph API 서버를 Custom Container로 배포해 **MGR API의 한계를 보완**, **이미지 분석 29.4s → 0.9s**.

**저장소** · [github.com/ddongjunn/go-ceph](https://github.com/ddongjunn/go-ceph)

<details markdown="1">
<summary>자세히 보기</summary>

**Problem**

Ceph MGR API만으로 클러스터 기능을 활용하는 구조에서 RBD 이미지 수 증가 시 리스트 조회가 3.2s까지 증가했고, 데이터 저장 위치 분석 기능 등 MGR API가 제공하지 않는 기능은 서비스 레벨에서 구현할 수 없는 한계가 있었습니다.

**Analyze**

네이티브 라이브러리(librados) 직접 접근이 가능한 독립 서비스가 필요했습니다. 클러스터 내장 플러그인 방식(MGR Module)은 Python 전용이고 배포 절차가 복잡해 개발·운영 부담이 컸고, 독립 컨테이너로 배포하는 방식이 언어 제약 없이 클러스터와 느슨하게 연결되어 더 적합하다고 판단했습니다.

**Action**

- go-ceph 기반 API 서버를 구현해 클러스터에 Custom Container 형태로 배포
- 이미지-디스크 매핑 조회 시 실제 할당된 객체만 선별해 불필요한 I/O 제거
- 객체의 저장 위치 분석 단계를 워커 풀 기반 병렬 처리로 분산 수행

**Result**

- RBD 이미지 리스트 조회 **3.2s → 1.2s** 단축
- 11.8 GiB 이미지(약 128,000 객체) 저장 위치 분석 **29.4s → 0.9s** 단축

</details>

### Ansible 기반 Ceph 클러스터 배포 자동화

`오케스트로 · 2025.03 ~ 2025.04`

설치·설정·서비스 구성 전 과정을 Ansible로 선언적 표준화해 클러스터 구축을 **반나절 → 3노드 기준 5~10분**으로 단축.

**저장소** · [github.com/ddongjunn/ceph-ansible](https://github.com/ddongjunn/ceph-ansible)

<details markdown="1">
<summary>자세히 보기</summary>

**Problem**

설정 변경이 잦은 초기 개발 단계에서 Ceph 클러스터를 반복 생성·폐기해야 했지만 설치부터 노드별 설정·서비스 구성까지 수작업으로 진행되어 클러스터 하나 구축에 반나절 이상 소요됐고 환경 편차가 발생했습니다.

**Analyze**

설치 단계만 자동화하는 것으로는 부족했고, 클러스터 설정·서비스 배포·리소스 구성까지 코드로 표준화할 필요가 있었습니다. 노드 구성·네트워크·서비스 옵션을 환경별로 유연하게 변경할 수 있어야 했기 때문에 변수·템플릿 기반으로 전 과정을 선언적으로 관리할 수 있는 Ansible 구조를 채택했습니다.

**Action**

- bootstrap/common/services/health_check 역할 단위로 배포·삭제·상태 검증 플로우를 분리
- group_vars/all.yml 전역 변수로 노드 구성·네트워크·서비스 옵션을 선언적으로 관리
- Jinja2 템플릿으로 서비스 spec을 동적 생성해 환경별 구성 변경 대응
- cephctl.sh 단일 명령으로 전체 배포·삭제 플로우 실행

**Result**

- 반나절 이상 걸리던 구축 시간을 **3노드 기준 5~10분**으로 단축
- 환경 편차 없이 동일한 구성의 클러스터를 반복 생성 가능한 구조 확보

</details>

### PaaS 플랫폼 — CI/CD 파이프라인 기능 개발

`오케스트로 · 2024.09 ~ 2024.12`

FreeMarker 기반 Pipeline Template과 Jenkins API를 활용해 사용자 입력에 따라 **Job을 동적 생성**하고, 신규 빌드 도구는 템플릿 확장만으로 대응할 수 있는 구조를 구축했습니다.

<details markdown="1">
<summary>자세히 보기</summary>

**Problem**

PaaS 플랫폼에서 사용자가 빌드 도구·저장소 타입·주소 등을 조합해 파이프라인을 구성할 수 있어야 했지만, 입력 조합이 다양해 정적으로 정의된 Jenkins Job으로는 모든 케이스를 수용하기 어려웠습니다.

**Analyze**

공통 파이프라인 구조를 템플릿으로 정의하고 사용자 입력값을 변수로 주입해 Job을 동적으로 생성하는 구조가 적합하다고 판단했습니다.

**Action**

- FreeMarker 기반 Pipeline Template 구조를 설계하고 Jenkins API를 통해 Job 동적 생성·갱신 기능 구현
- 저장소 타입 및 빌드 설정 입력값 검증 로직을 추가해 파이프라인 구성 표준화

**Result**

- 사용자 입력 기반으로 Jenkins Job을 동적으로 생성·실행하는 구조 확보
- 신규 빌드 도구·저장소 타입 추가 시 공통 Pipeline Template 확장만으로 대응 가능한 구조 구축

</details>

### ERP 연동 공통 인증 모듈 및 결재 정합성 개선

`코비젼 · 2023.11 ~ 2024.05`

AOP 공통 인증 모듈로 중복 인증 로직을 일원화하고, ERP 응답 기반으로 결재 흐름을 제어해 **외부 시스템과의 데이터 정합성** 개선했습니다.

<details markdown="1">
<summary>자세히 보기</summary>

**Problem**

전자결재 상신 시 외부 ERP API를 호출하는 과정에서 API별 인증 로직이 중복 구현되어 유지보수 비용이 증가하고 있었습니다. 또한 ERP 응답 성공 여부와 무관하게 결재가 승인으로 진행되는 구조로, 외부 시스템과의 데이터 정합성이 깨질 수 있는 문제가 있었습니다.

**Analyze**

인증 로직은 API마다 개별 처리하기보다 횡단 관심사(cross-cutting concern)로 분리하는 것이 확장성과 유지보수 측면에서 더 적합하다고 판단했습니다. 정합성 문제는 기존 결재 프로세스를 전면 수정할 경우 영향 범위가 커 리스크가 높았기 때문에, ERP 응답 결과를 기준으로 승인 흐름을 제어하는 방식으로 개선했습니다.

**Action**

- AOP 기반 공통 인증 모듈로 API별 중복된 로그인·인증 체크 로직을 일원화
- ERP 응답 코드 기준으로 예외 처리 규칙을 정의하고, 결재 승인 흐름을 조건 기반으로 제어
- API 호출 로깅을 추가해 장애 발생 시 요청·응답 단위로 추적 가능하도록 개선

**Result**

- 신규 ERP API 추가 시 공통 인증 모듈 재사용 가능 구조 확보
- ERP 호출 실패 시 해당 결재 단계가 실패 처리되도록 변경해 후속 결재 진행 차단 및 데이터 정합성 문제 개선

</details>

### APM 도입 및 쿼리 튜닝을 통한 성능 최적화

`코비젼 · 2023.05 ~ 2023.08`

Scouter APM과 실행계획(EXPLAIN) 분석으로 병목 조인을 식별하고 인덱스를 재설계해 **API 응답 시간 3.375s → 0.016s**로 개선했습니다.

<details markdown="1">
<summary>자세히 보기</summary>

**Problem**

SaaS 기반 그룹웨어 환경에서 다수 고객사의 전자결재 데이터가 단일 시스템에 누적되면서, 특정 전자결재 설정 API에서 응답 지연이 발생했습니다. 레거시 기반의 다중 조인 쿼리가 포함되어 있어, 코드 분석만으로는 병목 지점을 파악하기 어려운 상황이었습니다.

**Analyze**

추측 기반으로 쿼리를 수정할 경우 원인 분석 없이 수정 범위만 커질 수 있어, API 내부 구간별 수행 시간을 먼저 측정하는 것이 필요하다고 판단했습니다. SQL 단위 수행 시간을 추적할 수 있는 APM을 도입하고, 실행계획(EXPLAIN) 기반으로 병목 구간을 분석하는 방향으로 접근했습니다.

**Action**

- Scouter APM을 도입해 API 요청 구간별 및 SQL 단위 수행 시간을 가시화
- 실행계획(EXPLAIN) 비교 분석을 통해 병목 조인 구간 식별 및 다중 조인 구조 재구성
- 실행계획상 Full Table Scan · Using Temporary · Using Filesort 발생 원인을 분석하고 카디널리티 기반으로 인덱스 재설계

**Result**

- 해당 API 응답 시간 **3.375s → 0.016s** 개선
- Full Table Scan 기반 조회를 Index Range Scan 중심 구조로 개선

</details>

### 데모 사이트 발급 자동화 시스템

`코비젼 · 2022.09 ~ 2022.12`

데모 계정 발급 과정을 셀프 서비스로 자동화하고, 메일 발송을 **트랜잭션 커밋 후 비동기 이벤트로 분리**해 외부 시스템 실패를 격리했습니다.

<details markdown="1">
<summary>자세히 보기</summary>

**Problem**

기존에는 데모 요청을 영업팀과 개발팀이 수동으로 처리하고 있었고, 셀프 서비스 기반으로 전환하는 과정에서 계정 생성·메일 발송이 하나의 트랜잭션에 묶여 있어 외부 메일 시스템 실패 시 계정 생성 전체가 롤백되는 문제가 있었습니다.

**Analyze**

외부 시스템 실패가 핵심 도메인 로직의 성공 여부까지 영향을 주는 구조였기 때문에, 메일 발송을 메인 트랜잭션 및 요청 흐름으로부터 분리하는 방향으로 재설계했습니다.

**Action**

- 템플릿 설정·도메인 생성·백오피스 연동까지 계정 발급 전 과정을 자동화
- 트랜잭션 커밋 이후 이벤트를 발행하고, 비동기 이벤트 기반으로 메일 발송을 메인 트랜잭션과 분리
- 이벤트 처리 실패 시 별도 로그로 기록해 운영 추적 가능하도록 구성

**Result**

- 데모 계정 발급 과정을 셀프 서비스 기반으로 자동화
- 외부 메일 시스템 실패가 핵심 도메인 로직에 영향을 주지 않는 구조로 개선

</details>

### 동시성 제어 기반 장비 예약 관리 시스템

`코비젼 · 2022.06 ~ 2022.08`

`version` 컬럼 기반 낙관적 락을 직접 구현해 동시 요청 환경에서도 **이중 예약을 구조적으로 차단**했습니다.

<details markdown="1">
<summary>자세히 보기</summary>

**Problem**

장비 예약 성공 시 외부 ERP 시스템으로 예약 정보를 연동해야 했으며, 동일 장비에 대한 동시 예약 요청 시 이중 예약이 발생할 수 있는 구조였습니다. 예약 시스템 특성상 단 한 번의 이중 예약도 데이터 정합성 문제로 이어질 수 있어, 동시 요청 환경에서도 이를 보장할 수 있는 구조가 필요했습니다.

**Analyze**

동시 충돌 가능성은 낮더라도 정합성은 반드시 보장해야 하는 영역이었기 때문에, 요청 직렬화보다 충돌 시점에만 검증하는 방식이 적합하다고 판단했고, MyBatis 환경에서 version 컬럼 기반 낙관적 락을 직접 구현했습니다.

**Action**

- version 컬럼 기반 조건부 UPDATE와 affected row 검증으로 낙관적 락 구현
- 충돌 발생 시 예약 실패 처리 및 사용자 예외 응답 반환 구조 적용

**Result**

- 동일 장비에 대한 이중 예약 가능성을 구조적으로 차단
- 요청 직렬화 없이 동시 요청 환경에서도 예약 정합성을 보장하는 처리 흐름 확보

</details>

---

## 대외활동

- **Spring Boot 오픈소스 기여** — PR [#48347](https://github.com/spring-projects/spring-boot/pull/48347) (4.1.0-M1 반영). `MustacheResourceTemplateLoader` 템플릿 인코딩 처리 개선, public API 호환성을 고려한 설계 보완 후 머지.

---

## 교육

- 항해 플러스 백엔드 4기 (2024.03 ~ 2024.05)
- 학점은행제 · 정보통신공학 학사 (2022.10 ~ 2023.02)
- 비트캠프 · 웹개발 (2021.03 ~ 2021.09)
- 동양미래대학 · 정보통신공학과 전문학사 (2012.03 ~ 2021.02)
