# 🏠 HomeSweetHome - High-Performance Notification System

> **대규모 트래픽 처리를 위한 고성능 알림 시스템**
>
> MAU 300만 규모의 서비스를 가정하여 설계된 이커머스의 알림 도메인 프로젝트입니다.

## 📝 Introduction

**HomeSweetHome**은 단순한 알림 서비스를 넘어, 대규모 트래픽 상황에서도 안정적이고 빠르게 알림을 전송하는 것을 목표로 합니다.
Spring Boot와 SSE(Server-Sent Events)를 기반으로 구축되었으며, 단일 서버의 한계를 극복하기 위해 **이벤트 발행**(Notification)과 **전송**(SSE)을 분리하고 **Redis**를 도입하여 20만 명 이상의 대규모 접속을 수용할 수 있는 확장성 있는 아키텍처를 구현했습니다.

### Key Goals
*   **초고성능 실시간성**: 최대 20만 명의 SSE 동시 접속 유지 및 1초 이내의 지연 없는 알림 전송
*   **고가용성 처리**: 초당 2,000개 이상의 알림 생성 및 실시간 전송 처리
*   **대용량 벌크 전송**: 수만 명 이상의 유저를 대상으로 하는 대량 알림 전송 최적화

---

## 🛠 Tech Stack

- **Environment**: Java 21, Spring Boot
- **Notification (MVC)**: Spring Boot, Spring Data JPA
- **SSE Server (Reactive)**: Spring WebFlux, Spring Data Redis Reactive
- **DataBase** : MySQL, Redis, Flyway
- **Message Queue** : Kafka
- **Security & Auth**: Spring Security, JWT, OAuth2
- **DevOps & Monitoring**: Docker, Prometheus, Grafana
- **Testing**: JUnit5, Jacoco

---

## 🏗 Architecture & Modules

이 레포지토리는 아래와 같이 구성되어있습니다. 

| Module | Description |
| --- | --- |
| **backend-core** | 알림 이외의 공통 기능 서버 |
| **notification** | 알림 데이터를 생성하고 DB에 저장하며, 이벤트를 발행하는 서버 |
| **sse** | Reactive Stack(WebFlux) 기반의 SSE 전송 전용 서버 |
| **sse-mvc** | Servlet Stack(MVC) 기반의 SSE 서버 (비교 및 레거시 용도) |

### System Design
<img width="2960" height="1636" alt="Untitled-6" src="https://github.com/user-attachments/assets/bbc483f8-5d7b-40d2-9ea2-57f7a3a4787d" />

---

## 🚀 Performance & Technical Challenges

프로젝트를 진행하며 직면한 문제들과 이를 해결하여 얻은 성능 개선 성과입니다.

### 1. 트랜잭션 분리와 캐싱으로 성능 개선
*   **문제**: SSE 전송 등 긴 Network I/O가 트랜잭션 내에 포함되어 DB 커넥션 점유 시간이 길어지고 병목 현상 발생
*   **해결**: DB 작업이 필요한 지점만 `@Transactional`로 분리하고, 변경이 적은 알림 템플릿에 로컬 캐시 적용
*   **성과**: **P95 응답 시간 80% 이상 개선** (342ms → 67ms)
*   👉 [자세히 보기](https://github.com/dogyungkim/Homesweethome/wiki/[성능-개선]-1.-트랜잭션-분리와-캐싱으로-알림-생성-성능-개선)

### 2. 단체 알림 Bulk Insert 최적화
*   **문제**: JPA `IDENTITY` 전략의 한계로 인해 단체 알림 저장 시 N번의 Insert가 발생하여 대량 처리(5,000건 이상) 시 타임아웃 위험
*   **해결**: JDBC Template을 활용한 Bulk Insert 구현 및 `LAST_INSERT_ID()`를 이용한 ID 매핑 최적화
*   **성과**: 5,000건 기준 **저장 성능 약 60배 향상** (10.5초 → 0.17초)
*   👉 [자세히 보기](https://github.com/dogyungkim/Homesweethome/wiki/[성능-개선]-2.-단체-알림-개선)

### 3. SSE 서버 최적화 및 1.5만 동시 연결 달성
*   **문제**: Servlet 기반 Blocking I/O 모델의 한계로 4,000 연결 시 OOM 발생 및 전송 지연 발생
*   **해결**: **Spring WebFlux(Netty)** 마이그레이션, JVM 튜닝, JSON 직렬화 중복 연산 제거(Pre-serialization)
*   **성과**: **동시 연결 4,000개 → 15,000개 이상** 확대 및 전송 지연 시간 약 30% 단축
*   👉 [자세히 보기](https://github.com/dogyungkim/Homesweethome/wiki/[성능-개선]-3.-SSE-1만-동시-연결을-향해)

## 🏗 Structural Improvement

시스템의 유연성과 안정성을 높이기 위해 진행한 주요 구조 개선 사항입니다.

### 1. 서비스 간 결합도 해소 및 트랜잭션 무결성 확보
*   **문제**: 알림 서비스 직접 의존으로 인한 실패 전파 및 테스트 어려움, 트랜잭션 롤백 시 알림 발송 문제
*   **해결**: 도메인 이벤트 발행 구조와 `@TransactionalEventListener(AFTER_COMMIT)` 도입
*   **성과**: 서비스 간 결합도를 완전히 제거하고, DB 트랜잭션이 성공한 경우에만 비동기로 알림이 발송되도록 보장
*   👉 [자세히 보기](https://github.com/dogyungkim/Homesweethome/wiki/[구조-개선]-1.-다른-서비스와-결합도-낮추기)

### 2. 컴파일 타임 타입 안정성 강화
*   **문제**: Enum 타입과 데이터(Payload)의 수동 매핑으로 인한 런타임 에러 발생 위험
*   **해결**: `TemplateNotification` 인터페이스를 통한 알림 타입과 데이터의 캡슐화
*   **성과**: 잘못된 데이터 조합을 컴파일 시점에 차단하여 런타임 안정성 및 개발 생산성 향상
*   👉 [자세히 보기](https://github.com/dogyungkim/Homesweethome/wiki/[구조-개선]-2.-컴파일-타임에-알림-이벤트-객체-오류-찾기)

### 3. SSE 전송 서버 분리 및 분산 환경 설계
*   **문제**: SSE 연결 유지 비용(메모리/네트워크)이 비즈니스 로직 서버의 자원을 점유
*   **해결**: SSE 전송 책임만 갖는 별도의 서버로 분리하고 Redis Pub/Sub을 이용한 브로드캐스팅 구현
*   **성과**: 서버 역할별 자원 최적화(CPU vs Memory) 및 배포 시 사용자 연결 유지 전략 확보
*   👉 [자세히 보기](https://github.com/dogyungkim/Homesweethome/wiki/[구조-개선]-3.-SSE-별도의-서버로-분리하기-feat.Redis)

### 4. Kafka 도입을 통한 비동기 큐 한계 극복
*   **문제**: Spring 인메모리 큐의 락 경합으로 인한 성능 저하 및 서버 종료 시 메시지 유실 위험
*   **해결**: 외부 메시지 브로커인 **Kafka**를 도입하여 프로듀서와 컨슈머 분리
*   **성과**: 초당 수천 건의 요청에도 락 경합 없는 안정적인 처리 가능, 메시지 영속성 및 순서 보장 확보
*   👉 [자세히 보기](https://github.com/dogyungkim/Homesweethome/wiki/[구조-개선]-4.-비동기-큐-한계-극복하기-feat.kafka)

## 📂 Wiki Documentation

프로젝트를 개발하며 깊이 있게 탐구하고 정리한 기술 문서들입니다.

### 🍎 실시간 알림 & SSE (Server-Sent Events)
*   **[SSE는 어떻게 연결을 유지할까?](https://github.com/dogyungkim/Homesweethome/wiki/[개념-정리]-SSE-:-SSE는-어떻게-연결을-유지할까%3F)**: 커널 레벨의 TCP 연결 유지 메커니즘 분석
*   **[Spring MVC SseEmitter의 내부 동작](https://github.com/dogyungkim/Homesweethome/wiki/[개념-정리]-Spring-MVC-SseEmitter-:-1만-연결하면-스레드가-1만개-필요하다고%3F)**: `DeferredResult`를 활용한 Non-blocking 워커 스레드 관리와 메시지 직렬화 과정
*   **[Epoll: 리눅스 고성능 I/O의 핵심](https://github.com/dogyungkim/Homesweethome/wiki/[개념-정리]-Epoll)**: Select/Poll의 한계를 극복하는 Event-driven I/O 통지 모델과 RB-Tree 기반 관리 체계

### ⚙️ 시스템 기초 & 성능
*   **[쓰레드(Thread)의 탄생과 가상 메모리](https://github.com/dogyungkim/Homesweethome/wiki/[개념-정리]-Thread-1)**: 프로세스 격리를 위한 가상 메모리 기술과 쓰레드 등장의 배경
*   **[PCB vs TCB: 쓰레드가 가벼운 이유](https://github.com/dogyungkim/Homesweethome/wiki/[개념-정리]-Thread-2)**: 컨텍스트 스위칭 시 TLB Flush를 피하는 쓰레드의 구조적 이점과 스택 독립성의 필요성
*   **[유저 쓰레드와 커널 쓰레드](https://github.com/dogyungkim/Homesweethome/wiki/[개념-정리]-Thread-3)**: M:N 매핑 모델과 모드 전환(Mode Switch) 오버헤드 분석

### 💾 데이터베이스 & JPA
*   **[JPA IDENTITY 전략과 Bulk Insert](https://github.com/dogyungkim/Homesweethome/wiki/[개념-정리]-GenerationType.IDENTITY시-bulk-insert가-되지-않는-이유)**: Hibernate 영속성 컨텍스트의 ID 필요성과 JDBC 배치 insert가 비활성화되는 구조적 이유
