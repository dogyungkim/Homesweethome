# 🏠 HomeSweetHome - High-Performance Notification System

> **대규모 트래픽 처리를 위한 고성능 알림 시스템**
>
> 1만 동시 접속(SSE) 처리와 대용량 알림 생성을 목표로 설계된 분산 아키텍처 프로젝트입니다.

## 📝 Introduction

**HomeSweetHome**은 단순한 알림 서비스를 넘어, 대규모 트래픽 상황에서도 안정적이고 빠르게 알림을 전송하는 것을 목표로 합니다.
Spring Boot와 SSE(Server-Sent Events)를 기반으로 구축되었으며, 단일 서버의 한계를 극복하기 위해 **이벤트 발행(Notification)**과 **전송(SSE)**을 분리하고 **Redis**를 도입하여 확장성 있는 아키텍처를 구현했습니다.

### Key Goals
*   **실시간성**: SSE를 이용한 지연 없는 알림 전송
*   **대용량 처리**: 트랜잭션 최적화 및 Bulk Insert를 통한 알림 생성 성능 극대화
*   **확장성**: Redis Pub/Sub을 활용한 분산 환경 지원 (10k Connections 달성 목표)

---

## 🛠 Tech Stack

### Backend
*   **Language**: Java 21
*   **Framework**: Spring Boot 3.5.6 (MVC & WebFlux)
*   **Database**: MySQL 8.0, Redis (Pub/Sub), H2 (Local/Test)
*   **ORM**: JPA (Hibernate)

### DevOps & Tools
*   **Build**: Gradle
*   **Monitoring**: Actuator, Jaeger, Prometheus, Grafana
*   **Test**: JUnit5, Jacoco

---

## 🏗 Architecture & Modules

이 프로젝트는 기능과 역할에 따라 멀티 모듈로 구성되어 있습니다.

| Module | Description | Port |
| --- | --- | --- |
| **backend-core** | 공통 도메인 엔티티, 유틸리티, 설정 등을 포함하는 코어 모듈 | - |
| **notification** | 알림 데이터를 생성하고 DB에 저장하며, 이벤트를 발행하는 서비스 | 8080 |
| **sse** | Reactive Stack(WebFlux) 기반의 SSE 전송 전용 서버 | 8081 |
| **sse-mvc** | Servlet Stack(MVC) 기반의 SSE 서버 (비교 및 레거시 용도) | - |

### System Design
<!-- TODO: Architecture Diagram Here -->
> *[Structure] Architecture Diagram Recommendation: 서버 간의 통신 흐름(Core -> Redis -> SSE -> Client)을 시각화한 이미지를 여기에 추가하세요.*

---

## 🚀 Performance & Technical Challenges

프로젝트를 진행하며 직면한 문제들과 이를 해결하여 얻은 성능 개선 성과입니다. 더 자세한 내용은 [Wiki](https://github.com/dogyungkim/Homesweethome/wiki)에서 확인하실 수 있습니다.

### 1. 트랜잭션 분리와 캐싱으로 성능 개선
알림 생성 시 발생하는 DB 부하를 줄이기 위해 트랜잭션을 분리하고 자주 조회되는 데이터에 캐싱을 적용했습니다.
*   **문제**: 알림 생성 로직과 비즈니스 로직이 강하게 결합되어 전체 응답 속도 저하
*   **해결**: `@TransactionalEventListener`를 사용한 트랜잭션 분리 및 Redis Caching
*   **성과**: 응답 시간 **N% 단축** (구체적 수치 반영 필요)
*   👉 [자세히 보기](Homesweethome.wiki/[성능%20개선]%201.%20트랜잭션%20분리와%20캐싱으로%20알림%20생성%20성능%20개선.md)

<!-- TODO: Transaction Improvement Chart Here -->

### 2. 단체 알림 Bulk Insert 최적화
수천 명의 유저에게 동시에 알림을 보낼 때 발생하는 `N+1` 문제와 Insert 성능 저하를 해결했습니다.
*   **문제**: JPA `saveAll`의 성능 한계 (Identity 전략 사용 시 Batch Insert 불가)
*   **해결**: JDBC Template을 활용한 **Bulk Insert** 구현
*   **성과**: 1000건 기준 저장 속도 **약 10배 이상 향상**
*   👉 [자세히 보기](Homesweethome.wiki/[성능%20개선]%202.%20단체%20알림%20개선.md)

<!-- TODO: Bulk Insert vs Single Insert Chart Here -->

### 3. SSE 서버 분리 및 1만 동시 연결 달성
Tomcat 기반의 기존 서버에서 블로킹 이슈와 커넥션 고갈 문제를 해결하기 위해 Netty 기반의 WebFlux로 마이그레이션했습니다.
*   **문제**: Servlet 스레드 풀 한계로 다수의 SSE 연결 유지 시 리소스 고갈
*   **해결**: **Spring WebFlux** 도입 및 **Redis Pub/Sub**을 이용한 스케일 아웃 구조 설계
*   **성과**: **10,000 동시 연결(Concurrent Connections)** 테스트 통과 및 안정적인 메모리 사용량 확인
*   👉 [자세히 보기](Homesweethome.wiki/[성능-개선]-3.-SSE-1만-동시-연결을-향해.md)

<!-- TODO: SSE Load Test Result Graph Here -->
---

## 📂 Wiki Documentation
이 프로젝트의 기술적 고민과 해결 과정은 Wiki에 상세히 기록되어 있습니다.
*   [개념 정리: SSE는 어떻게 연결을 유지할까?](Homesweethome.wiki/[개념%20정리]%20SSE%20:%20SSE는%20어떻게%20연결을%20유지할까?.md)
*   [구조 개선: 다른 서비스와 결합도 낮추기](Homesweethome.wiki/[구조%20개선]%201.%20다른%20서비스와%20결합도%20낮추기.md)
*   [구조 개선: SSE 별도의 서버로 분리하기](Homesweethome.wiki/[구조%20개선]%203.%20SSE%20별도의%20서버로%20분리하기%20feat.Redis.md)
