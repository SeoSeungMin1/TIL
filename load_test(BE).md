# 백엔드 성능·동시성 취약점 분석

> 분석 기준: 2026-08-10 현재 `apps/backend` 소스 코드  
> 범위: Spring MVC REST API, MongoDB Repository/Document, 로컬 파일 저장소, Netty Socket.IO, AI 스트리밍, `loadtest/load-test.js`, `loadtest/ramp-up-test.js`  
> 주의: 이 문서는 정적 코드 분석 결과이다. 운영 MongoDB에 수동 생성된 인덱스가 있는지는 코드만으로 알 수 없으므로, 인덱스 관련 최종 판정에는 운영 DB의 `getIndexes()`와 `explain("executionStats")` 확인이 필요하다. 아래의 사용자 수와 데이터 수는 성능 수치가 아니라 부하 크기를 설명하기 위한 **예시**다.

## 1. 전체 요약

### 1.1 검증 결과

| 번호 | 문제 | 심각도 | 관련 기능 | 핵심 원인 | 실제 코드에서 확인 여부 |
|---:|---|---|---|---|---|
| 1 | 방 목록 전체 조회 + N+1 | 매우 높음 | `GET /api/rooms` | 무페이지네이션 `findAll()` 뒤 방별 생성자·참가자 개별 조회와 최근 메시지 `count` | **확인됨** |
| 2 | 메시지 조회 복합 인덱스 부재 | 매우 높음 | `fetchPreviousMessages`, 방 입장 초기 조회 | `room + timestamp` 필터/정렬을 쓰지만 `Message`에 복합 인덱스 선언 없음 | **확인됨(코드 기준)**. 운영 DB 수동 인덱스는 별도 확인 필요 |
| 3 | 메시지 사용자·파일 N+1 | 높음 | 이전 메시지 조회, 방 입장 | 메시지마다 발신자 `findById`, 파일 메시지마다 파일 `findById` | **확인됨** |
| 4 | 읽음 처리 반복 DB 작업 | 매우 높음 | 이전 메시지 조회, `markMessagesAsRead` | 메시지 ID마다 `findById → save`; 이미 읽은 메시지도 저장 | **확인됨** |
| 5 | 읽음·리액션 동시성 문제 | 매우 높음 | 읽음, 리액션 | 동일 메시지 문서를 read-modify-save하여 Lost Update 가능 | **확인됨** |
| 6 | 메시지 전송당 DB 접근 과다 | 매우 높음 | `chatMessage` | 세션·Rate Limit·사용자·방·메시지·집계·세션 갱신이 한 이벤트에 직렬 연결 | **확인됨** |
| 7 | Rate Limit 원자성 문제 | 매우 높음 | REST Rate Limit, `chatMessage` | 카운터를 조회한 뒤 애플리케이션에서 증가시켜 저장; Mongo 트랜잭션 어노테이션만으로 경쟁 방지 불가 | **확인됨** |
| 8 | 방 참가 처리 비효율 | 높음 | REST 방 참가, Socket.IO `joinRoom`/`leaveRoom` | 참가자 전체 사용자 조회와 전체 목록 방송. REST는 방 전체 저장 | **부분 확인됨**: REST 전체 저장은 확인, Socket.IO 참가 갱신은 `$addToSet`으로 이미 원자화됨 |
| 9 | 추가 인덱스 부재 | 높음 | 최근 메시지 count, 파일 다운로드/보기 권한 검사 | `messages.room/timestamp`, `messages.file`, `files.filename` 조회용 선언 인덱스 없음 | **확인됨(코드 기준)**. 실제 COLLSCAN 여부는 `explain` 필요 |
| 10 | 파일 업로드/다운로드 서버 점유 | 높음 | `/api/files/upload`, `/download`, `/view` | 기본 로컬 저장소에서 동기 디스크 복사 및 서버 경유 응답; Tomcat 최대 10 스레드 | **확인됨(기본 `local` 설정)** |
| 11 | AI 스트리밍 누적 전송 | 높음 | AI 멘션 응답 | 매 청크마다 누적 문자열 전체를 방 전체에 전송; 문자열 연결과 무제한 demand | **확인됨** |
| 12 | Socket.IO 확장성 문제 | 매우 높음 | 연결, 룸, 브로드캐스트 | accept backlog 10, `MemoryStoreFactory`, 로컬 `ConcurrentHashMap` 상태 | **확인됨** |

심각도는 발생 가능성, 요청당 증폭 정도, 데이터 정합성 위험, 현재 서버 제한을 함께 고려한 코드 리뷰 우선순위다. 측정 결과가 아니며 부하테스트 뒤 조정해야 한다.

### 1.2 관점별 분류

| 관점 | 해당 문제 | 연결 관계 |
|---|---|---|
| 조회 성능 | 1, 2, 3, 9 | 전체 조회, N+1, 정렬/필터 인덱스 부재가 MongoDB 읽기를 증폭한다. |
| DB 작업량 | 1, 3, 4, 6, 8 | 한 논리 요청이 여러 MongoDB round trip과 전체 문서 저장으로 확대된다. |
| 동시성 | 5, 7, 8 | read-modify-save는 Lost Update를 만들 수 있다. 8번의 Socket.IO `$addToSet` 자체는 예외다. |
| 파일 I/O | 9, 10 | 파일 메타데이터 권한 조회와 실제 디스크 전송 비용이 겹친다. |
| AI | 11 | 응답 길이에 따라 누적 문자열 생성 및 네트워크 전송량이 비선형적으로 커진다. |
| Socket.IO / 확장성 | 6, 8, 11, 12 | 이벤트당 DB 작업, 방 전체 방송, 단일 노드 상태와 작은 연결 대기열이 함께 한계를 만든다. |

---

## 2. 각 취약점 상세 분석

### 2.1 방 목록 전체 조회 + N+1

#### 1) 이 문제가 무엇인가?

N+1은 목록을 가져오는 쿼리 1번 뒤, 목록의 각 항목마다 추가 쿼리를 N번 수행하는 패턴이다. 이 프로젝트는 방을 전부 한 번에 읽고 각 방의 생성자 1회, 참가자 수만큼 사용자 조회, 최근 30분 메시지 count 1회를 실행한다. 따라서 단순한 `1 + N`보다 실제로는 `1 + Σ(1 + 방별 참가자 수 + 1)`에 가깝다.

#### 2) 실제 코드 위치

- API: `src/main/java/com/ktb/chatapp/controller/RoomController.java` — `RoomController#getAllRooms`
- 서비스: `src/main/java/com/ktb/chatapp/service/RoomService.java` — `getAllRooms`, `mapToRoomResponse`
- Repository: `RoomRepository#findAll`, `UserRepository#findById`, `MessageRepository#countRecentMessagesByRoomId`
- 집계: `RecentMessageCounter#countRecentMessages`
- Document: `Room`(`rooms`), `User`(`users`), `Message`(`messages`)

```java
roomRepository.findAll().stream()
    .map(room -> mapToRoomResponse(room, name));

creator = userRepository.findById(room.getCreator()).orElse(null);
participants = room.getParticipantIds().stream()
    .map(userRepository::findById).toList();
recentMessageCounter.countRecentMessages(room.getId());
```

#### 3) 현재 코드 실행 흐름

`GET /api/rooms` → `RoomController#getAllRooms` → `RoomService#getAllRooms` → `RoomRepository#findAll` → JVM에서 모든 방 매핑 → 방마다 생성자 조회 → 방마다 참가자별 사용자 조회 → 방마다 최근 메시지 count → JVM에서 생성일 역순 정렬 → 전체 응답.

#### 4) 정확히 어디가 문제인가?

방 수를 `R`, 각 방 참가자 수 합계를 `P`라고 하면 정상 응답에서 최소 `1 + 2R + P`회의 MongoDB 작업이 발생할 수 있다. 생성자가 참가자 집합에도 있으면 같은 사용자를 중복 조회한다. 페이지네이션이 없고 정렬도 DB가 아니라 JVM에서 수행한다. 각 count는 별도 네트워크 왕복이며 2·9번의 인덱스 부재 영향도 받는다. 컨트롤러의 10초 `Cache-Control`은 브라우저 지시일 뿐 서버 공용 캐시가 아니며 `Last-Modified`를 매번 현재 시각으로 내려 서버 계산을 제거하지 않는다.

#### 5) 데이터가 많아지면 어떻게 되는가?

**예시:** 방 10개, 방당 참가자 10명이면 최대 약 121회(`1 + 20 + 100`)의 DB 작업이다. 방 10,000개, 방당 100명이면 약 1,020,001회가 될 수 있고, 모든 방 객체·DTO를 메모리에 유지한 채 정렬하고 직렬화한다. 실제 횟수와 시간은 데이터 중복, 드라이버, 캐시, 인덱스에 따라 달라진다.

#### 6) 서버에는 어떤 영향을 주는가?

응답시간 증가, RPS 감소, MongoDB operation/CPU 증가, DB 네트워크 왕복 증가, 애플리케이션 CPU·메모리·GC 증가, 큰 JSON 응답에 따른 네트워크 사용량과 Tomcat thread 점유가 발생한다.

#### 7) 부하테스트에서 어떻게 나타날까?

VU 증가 → `GET /api/rooms` 동시 호출 증가 → 요청당 `findById`/count 폭증 → MongoDB operation과 `docsExamined` 증가 → p95/p99 급등 → Tomcat 10개 스레드 및 DB 커넥션 대기 → RPS 정체·오류 증가.

#### 8) 어떻게 부하테스트해야 하는가?

- 대상: `GET /api/rooms`
- 데이터: 방 10/1,000/10,000 단계, 방당 참가자 1/10/100, 최근 메시지 포함
- 부하: 1 → 10 → 25 → 50 VU 계단식 증가, 각 단계 3분과 1분 휴지
- 확인: p50/p95/p99, RPS, error rate, 응답 크기, CPU/memory/GC, MongoDB command 수, query latency, `docsExamined`, `keysExamined`, 네트워크
- `ramp-up-test.js`는 방 생성 때 `fetchRoomsList()`로 이 API를 호출하므로 징후 확인은 가능하지만, 방 목록만 격리하고 방 수를 고정하는 별도 HTTP 테스트가 필요하다.

#### 9) 개선 방법

1순위: DB 정렬과 커서/페이지 기반 조회를 도입하고 응답 필드를 제한한다. 2순위: 페이지 내 creator/participant ID를 모아 `findAllById` 한 번으로 batch 조회하고 Map으로 조립한다. 3순위: 최근 count를 매번 계산하지 말고 이벤트 기반 카운터/별도 집계 컬렉션/짧은 서버 캐시를 사용하며 중복 생성자 조회를 제거한다.

#### 10) 개선 전 / 개선 후 예상 구조

```text
개선 전: 전체 방 조회 → 방마다 creator → 참가자마다 user → 방마다 count → JVM 정렬
개선 후: 정렬된 방 한 페이지 조회 → 사용자 ID 일괄 조회 → 집계 일괄 조회/캐시 → 응답
```

#### 11) 개선 효과를 어떻게 검증할까?

같은 DB 스냅샷·인덱스·VU·시간으로 Before/After를 실행한다. p95/p99와 RPS뿐 아니라 요청 1건당 MongoDB operation 수, 응답 bytes, CPU, heap/GC를 비교한다. 목표는 방 수가 늘어도 요청당 쿼리 수가 페이지 크기에 비례해 폭증하지 않는지 확인하는 것이다.

#### 12) 핵심 정리

> 핵심: 방 목록 한 번이 실제로는 방·참가자 수에 비례하는 다수의 DB 호출이다.  
> 페이지네이션, 사용자 batch 조회, 최근 메시지 집계 최적화가 함께 필요하다.

---

### 2.2 메시지 조회 복합 인덱스 부재

#### 1) 이 문제가 무엇인가?

복합 인덱스는 여러 필드를 정해진 순서로 묶은 인덱스다. 이 프로젝트의 이전 메시지 조회는 `room = ?`, `timestamp < ?`, `timestamp DESC`를 동시에 사용하므로 일반적으로 `{ room: 1, timestamp: -1 }` 형태가 핵심이다. 현재 `Message`에는 인덱스 선언이 없다.

#### 2) 실제 코드 위치

- 이벤트: `MessageFetchHandler#handleFetchMessages`, `RoomJoinHandler#handleJoinRoom`
- 조회: `MessageLoader#loadMessagesInternal`
- Repository: `MessageRepository#findByRoomIdAndTimestampBefore`
- Document: `Message`의 `@Field("room") roomId`, `timestamp`
- 설정: `application.properties`의 `spring.data.mongodb.auto-index-creation=true`지만 생성할 `Message` 인덱스 선언이 없음

```java
Pageable pageable = PageRequest.of(0, limit, Sort.by("timestamp").descending());
messageRepository.findByRoomIdAndTimestampBefore(roomId, before, pageable);
```

#### 3) 현재 코드 실행 흐름

`fetchPreviousMessages` 또는 `joinRoom` → `MessageLoader` → `messages`에서 room/timestamp 조건 조회 → timestamp 역순 정렬 → 최대 30개 반환. 반환형이 `Page`이므로 Spring Data가 `hasNext/total` 계산을 위해 count 쿼리를 추가 실행할 가능성도 있다.

#### 4) 정확히 어디가 문제인가?

코드에 `@CompoundIndex`, `MongoTemplate.ensureIndex`, 마이그레이션 스크립트가 없다. 운영 DB에도 인덱스가 없다면 MongoDB는 많은 문서를 검사하고 정렬을 메모리에서 수행할 수 있다. `_id` 기본 인덱스는 room/timestamp 조건에 도움을 주지 않는다. 단, 수동 생성 인덱스 존재 여부와 실제 COLLSCAN은 소스만으로 확정할 수 없다.

#### 5) 데이터가 많아지면 어떻게 되는가?

**예시:** 전체 100개 메시지에서는 차이가 작지만 1,000,000개 중 특정 방의 최근 30개를 찾을 때 적절한 인덱스가 없으면 30개만 반환하려고도 훨씬 많은 문서를 검사할 수 있다. 오래된 `before`로 갈수록 스캔 범위가 달라질 수 있다.

#### 6) 서버에는 어떤 영향을 주는가?

MongoDB CPU·메모리·디스크 I/O 증가, 응답 p95/p99 증가, RPS 감소, Socket.IO event 처리 지연이 발생한다.

#### 7) 부하테스트에서 어떻게 나타날까?

동시 입장/스크롤 증가 → room/timestamp 조회 증가 → `docsExamined`가 반환 건수보다 크게 증가 → DB CPU와 query latency 상승 → `previousMessagesLoaded` 지연 → 연결 직후 체감 TTI 악화.

#### 8) 어떻게 부하테스트해야 하는가?

- 대상: Socket.IO `fetchPreviousMessages`; 보조 대상 `joinRoom`
- 데이터: 총 10만/100만 메시지, 여러 방에 분산, 방별 최신·중간·오래된 cursor
- 부하: 10 → 50 → 100 동시 소켓, 사용자당 2초마다 페이지 요청, 단계당 3분
- 확인: 이벤트 RTT p50/p95/p99, 성공률, MongoDB `executionStats`, `docsExamined`, `keysExamined`, sort stage, CPU/I/O
- 기존 두 스크립트 모두 조회 이벤트를 보내지만 첫 페이지만 조회한다. cursor 깊이와 조회 빈도를 통제하는 별도 테스트가 필요하다.

#### 9) 개선 방법

1순위: 실제 쿼리에 맞는 `{ room: 1, timestamp: -1 }` 복합 인덱스를 명시적으로 관리한다. 2순위: `Page` 대신 `Slice` 또는 `limit + 1` 방식으로 불필요한 total count를 피한다. 3순위: `explain`으로 필드명 `room`, 정렬 방향, 범위 조건에 맞는 winning plan을 확인하고 필요 시 projection을 적용한다.

#### 10) 개선 전 / 개선 후 예상 구조

```text
개선 전: room 조건 후보 탐색/스캔 → timestamp 정렬 → 30개 선택 (+ count 가능)
개선 후: room+timestamp 인덱스 seek → 정렬된 인덱스에서 31개 읽기 → hasMore 계산
```

#### 11) 개선 효과를 어떻게 검증할까?

Before/After에 동일한 컬렉션과 cursor를 사용한다. `COLLSCAN → IXSCAN`, `docsExamined/returned`, `keysExamined`, execution time, 이벤트 p95/p99, DB CPU·I/O를 비교한다. 인덱스 크기와 쓰기 비용도 함께 기록한다.

#### 12) 핵심 정리

> 핵심: 이전 메시지 쿼리는 room 필터와 timestamp 범위·정렬을 동시에 사용한다.  
> 코드에는 이를 지원하는 복합 인덱스가 없어 운영 DB의 실행 계획 검증이 필수다.

---

### 2.3 메시지 사용자·파일 N+1

#### 1) 이 문제가 무엇인가?

메시지 목록 1회를 읽은 뒤 각 메시지 발신자를 따로 읽고, 파일 메시지라면 파일도 따로 읽는 N+1이다. 같은 발신자가 연속으로 보낸 메시지도 캐시 없이 반복 조회한다.

#### 2) 실제 코드 위치

- `MessageLoader#loadMessagesInternal`, `findUserById`
- `MessageResponseMapper#mapToMessageResponse`
- `UserRepository#findById`, `FileRepository#findById`
- Document: `Message.sender`/`Message.file`, `User`, `File`

```java
sortedMessages.stream().map(message -> {
    var user = userRepository.findById(message.getSenderId());
    return messageResponseMapper.mapToMessageResponse(message, user.orElse(null));
});
// mapper 내부
Optional.ofNullable(message.getFileId()).flatMap(fileRepository::findById);
```

#### 3) 현재 코드 실행 흐름

메시지 페이지 조회 → 각 메시지의 sender 조회(AI/system은 sender가 null이면 생략) → 각 파일 메시지의 file 조회 → DTO 조립 → `previousMessagesLoaded`.

#### 4) 정확히 어디가 문제인가?

메시지 수 `M`, 파일 메시지 수 `F`일 때 페이지 쿼리 외에 최대 `M + F`회 조회한다. 기본 limit은 30이므로 최대 45회 추가 조회가 아니라, 모든 30개가 파일이면 60회 추가 조회다. 동일 사용자·파일 중복 제거도 없다. 이 흐름에는 4번 읽음 갱신의 최대 60회 find/save까지 추가된다.

#### 5) 데이터가 많아지면 어떻게 되는가?

**예시:** 30개 페이지에서 발신자 조회 30회와 파일 조회 10회가 추가된다. 100명이 동시에 입장하면 동일 데이터에 대해 약 4,000회의 보조 조회가 생길 수 있다. 실제 값은 메시지 유형에 따라 달라진다.

#### 6) 서버에는 어떤 영향을 주는가?

응답시간 증가, MongoDB operation과 네트워크 왕복 증가, RPS 감소, Socket.IO worker 지연, DB 커넥션 대기 증가가 발생한다.

#### 7) 부하테스트에서 어떻게 나타날까?

동시 `joinRoom`/fetch 증가 → 메시지 페이지 수보다 user/file find가 훨씬 많이 증가 → 이전 메시지 p95 상승 → 입장 TTI 상승 → DB operation이 소켓 사용자 수에 선형 이상으로 증가.

#### 8) 어떻게 부하테스트해야 하는가?

- 대상: `fetchPreviousMessages`, `joinRoom`
- 데이터: 방당 1,000개 메시지, 페이지당 고유 발신자 비율 10%/100%, 파일 비율 0%/50%/100%
- 부하: 1/25/50/100 동시 요청, 5분
- 확인: 이벤트 p50/p95/p99, MongoDB find operation 수, DB 네트워크, CPU, error rate
- 기존 스크립트로 징후 확인 가능하나 파일 비율과 발신자 분포를 고정하는 별도 seed/시나리오가 필요하다.

#### 9) 개선 방법

1순위: 한 페이지의 senderId와 fileId를 distinct로 모아 `findAllById` 두 번으로 일괄 조회한다. 2순위: 요청 범위 또는 짧은 TTL 사용자 캐시를 적용한다. 3순위: 자주 표시하는 발신자/파일 요약을 메시지에 비정규화하되 변경 동기화 정책을 명확히 한다.

#### 10) 개선 전 / 개선 후 예상 구조

```text
개선 전: 메시지 30개 → user 최대 30회 → file 최대 30회
개선 후: 메시지 30개 → 고유 user ID batch 1회 → 고유 file ID batch 1회 → Map 조립
```

#### 11) 개선 효과를 어떻게 검증할까?

같은 페이지와 동시성에서 Before/After의 요청당 DB find 수, 이벤트 p95/p99, DB 네트워크·CPU를 비교한다. 응답 JSON이 동일한지도 계약 테스트로 확인한다.

#### 12) 핵심 정리

> 핵심: 메시지 페이지 크기만큼 사용자·파일 조회가 반복된다.  
> 페이지 안의 ID를 모아 두 번의 batch 조회로 바꾸는 것이 핵심이다.

---

### 2.4 읽음 처리 반복 DB 작업

#### 1) 이 문제가 무엇인가?

여러 메시지를 읽음 처리할 때 DB가 지원하는 bulk update를 쓰지 않고 Java 반복문에서 각 메시지를 읽고 수정한 뒤 저장한다. 이는 논리적으로 한 번인 “30개 읽음”을 최대 60회의 DB 작업으로 확대한다.

#### 2) 실제 코드 위치

- `MessageReadStatusService#updateReadStatus`
- 호출: `MessageLoader#loadMessagesInternal`, `MessageReadHandler#handleMarkAsRead`
- Repository: `MessageRepository#findById`, `save`
- Document: `Message.readers[]` (`Message.MessageReader`)

```java
for (String messageId : messageIds) {
    var message = messageRepository.findById(messageId);
    // readers 검사·추가
    messageRepository.save(message);
}
```

#### 3) 현재 코드 실행 흐름

이전 메시지 30개 조회 또는 `markMessagesAsRead` 수신 → ID 반복 → 각 ID find → readers 전체 순회 → 필요 시 append → 필요 여부와 무관하게 save → 방 전체 `messagesRead` 방송.

#### 4) 정확히 어디가 문제인가?

`K`개 ID마다 최대 `2K` round trip이다. `alreadyRead`여도 51행의 `save`는 실행된다. 이전 메시지를 단순 조회하는 것만으로도 자동 읽음 저장이 발생하고, 클라이언트가 별도 읽음 이벤트까지 보내면 중복 작업이 가능하다. 또한 서비스가 예외를 잡아 삼키므로 일부 메시지만 갱신된 상태에서도 상위 핸들러가 성공 방송을 할 수 있다.

#### 5) 데이터가 많아지면 어떻게 되는가?

**예시:** 페이지 30개를 1,000명이 동시에 읽으면 최대 60,000회의 DB 작업이 추가된다. 메시지 readers 배열이 커질수록 매번 전체 문서 전송·역직렬화·저장 비용도 커진다.

#### 6) 서버에는 어떤 영향을 주는가?

MongoDB 쓰기 부하, 디스크 I/O, 네트워크 왕복, 응답시간과 p99 증가, RPS 감소, 데이터 불일치(부분 성공), CPU와 GC 증가가 발생한다.

#### 7) 부하테스트에서 어떻게 나타날까?

VU 증가 → fetch/mark 이벤트 증가 → 메시지당 find/save 증가 → MongoDB write operations와 latency 증가 → `previousMessagesLoaded` 및 `messagesRead` 지연 → write queue/CPU 상승 → 오류율 증가.

#### 8) 어떻게 부하테스트해야 하는가?

- 대상: `markMessagesAsRead`와 `fetchPreviousMessages`
- 데이터: 한 방 10,000개 메시지; 읽지 않음/이미 읽음 두 조건
- 부하: 10 → 50 → 200 사용자, 이벤트당 1/30/100 IDs, 5분
- 확인: 이벤트 p50/p95/p99, MongoDB find/update 수, write latency, bytes written, CPU/I/O, 중복 reader 여부, 성공 방송과 실제 DB 상태 일치
- 기존 스크립트는 수신 메시지를 1개씩 읽고 fetch도 수행하므로 징후는 잘 드러난다. 30/100개 batch 비교는 별도 테스트가 필요하다.

#### 9) 개선 방법

1순위: 조건부 `$addToSet`/`$push`와 `updateMulti`/bulkWrite로 여러 ID를 한 번 또는 소수의 명령으로 갱신한다. 2순위: 이미 읽은 경우를 쿼리 조건으로 제외하고 no-op save를 없앤다. 3순위: 메시지별 readers 배열 대신 사용자·방별 read cursor(마지막 읽은 timestamp/messageId) 모델을 검토한다.

#### 10) 개선 전 / 개선 후 예상 구조

```text
개선 전: ID마다 find → readers 검사 → save
개선 후: 권한 검증 → 조건부 bulk update 1회(또는 방별 read cursor 1회) → 결과 기준 방송
```

#### 11) 개선 효과를 어떻게 검증할까?

동일 ID 수와 VU에서 Before/After의 update 명령 수, write bytes, p95/p99, DB CPU/I/O를 비교한다. 이미 읽은 재요청이 쓰기를 발생시키지 않는지, 부분 실패 시 성공으로 방송하지 않는지도 확인한다.

#### 12) 핵심 정리

> 핵심: 읽음 ID 하나마다 find와 save를 반복하고 no-op도 저장한다.  
> 조건부 bulk update 또는 방별 read cursor로 DB 작업 수를 줄여야 한다.

---

### 2.5 읽음·리액션 동시성 문제

#### 1) 이 문제가 무엇인가?

read-modify-save는 문서를 읽고 애플리케이션 메모리에서 바꾼 뒤 전체 문서를 저장하는 방식이다. 두 요청이 같은 이전 버전을 읽으면 나중 저장이 먼저 저장된 변경을 덮는 Race Condition, 즉 Lost Update가 생길 수 있다.

#### 2) 실제 코드 위치

- 읽음: `MessageReadStatusService#updateReadStatus`
- 리액션: `MessageReactionHandler#handleMessageReaction`
- 도메인 변경: `Message#addReaction`, `removeReaction`
- Repository: `MessageRepository#findById`, `save`
- Document: 동일 `Message`의 `readers`, `reactions`

```java
Message message = messageRepository.findById(id).orElse(...);
message.addReaction(reaction, userId); // 또는 readers.add(...)
messageRepository.save(message);      // 문서 전체 교체 성격
```

#### 3) 현재 코드 실행 흐름

요청 A read → 요청 B read → A가 readers 또는 reactions 변경 후 save → B가 자신의 오래된 복사본을 변경 후 save → A 변경이 사라질 수 있음 → 각 핸들러는 자신이 만든 응답을 방송.

#### 4) 정확히 어디가 문제인가?

낙관적 잠금용 `@Version`이 없고, `$addToSet`, `$pull`, array filter 같은 필드 단위 원자 업데이트를 사용하지 않는다. 읽음과 리액션은 서로 다른 필드라도 동일 문서를 저장하므로 상호 덮어쓰기도 가능하다. 방송된 상태와 최종 DB 상태가 달라질 수 있다.

#### 5) 데이터가 많아지면 어떻게 되는가?

**예시:** 인기 메시지 하나에 100명이 동시에 읽음과 👍 추가를 수행하면 여러 요청이 같은 버전을 읽을 확률이 커진다. 최종 readers/reactions 수가 성공 이벤트 수보다 작을 수 있다. 정확한 유실률은 타이밍에 의존하므로 코드만으로 수치화할 수 없다.

#### 6) 서버에는 어떤 영향을 주는가?

데이터 유실, Race Condition, Lost Update, 클라이언트 간 상태 불일치, 재시도에 따른 DB 부하가 발생한다.

#### 7) 부하테스트에서 어떻게 나타날까?

동일 messageId 집중 동시 요청 → 모두 성공 이벤트 수신 → 테스트 종료 후 DB의 고유 reader/reaction 사용자 수가 성공 사용자 수보다 작음 → 방송 스냅샷이 앞뒤로 되돌아가는 현상.

#### 8) 어떻게 부하테스트해야 하는가?

- 대상: `markMessagesAsRead`, `messageReaction`
- 데이터: 참가자 100~1,000명, 단일 target message와 여러 target 대조군
- 부하: barrier로 50/100/500 동시 emit, add/read 혼합을 100회 반복
- 확인: 성공 ack/event 수, DB의 고유 readers/reactions 수, 중복·유실, p95/p99, MongoDB write conflict/operation
- 기존 스크립트는 10% 랜덤 반응이라 재현성이 낮다. 동일 messageId에 동시 집중하는 별도 정합성 부하테스트가 반드시 필요하다.

#### 9) 개선 방법

1순위: 읽음은 `$addToSet`, 리액션 추가는 `reactions.<emoji>`에 `$addToSet`, 제거는 `$pull`을 사용하는 단일 원자 update로 바꾼다. 2순위: 사용자 입력 emoji를 필드 경로로 쓸 때 Mongo 키 제약/인젝션을 피하도록 reaction 배열 스키마와 검증을 설계한다. 3순위: 복합 변경에 `@Version` 낙관적 잠금과 제한된 재시도를 사용하고 DB 결과를 기준으로 방송한다.

#### 10) 개선 전 / 개선 후 예상 구조

```text
개선 전: 두 요청이 같은 문서 read → 각자 변경 → 전체 save → 마지막 저장만 잔존 가능
개선 후: 요청별 조건부 atomic update → MongoDB가 배열 원소 단위 반영 → 확정 결과 방송
```

#### 11) 개선 효과를 어떻게 검증할까?

동일한 barrier·사용자·반복 횟수로 Before/After를 실행한다. 성공 요청 수와 최종 고유 원소 수가 일치하는지 최우선으로 보고, p95/p99와 write operation/재시도 수도 비교한다.

#### 12) 핵심 정리

> 핵심: 읽음과 리액션이 같은 메시지 문서를 읽고 전체 저장해 서로의 변경을 덮을 수 있다.  
> 필드 단위 MongoDB 원자 연산과 DB 확정 결과 기반 방송이 필요하다.

---

### 2.6 메시지 전송당 DB 접근 과다

#### 1) 이 문제가 무엇인가?

사용자가 텍스트 메시지 한 건을 보내는 간단한 동작이 여러 저장소 조회·저장·count로 이어진다. 각 작업이 직렬이면 가장 느린 구간들이 누적되고, 메시지 RPS보다 DB operation 증가율이 훨씬 커진다.

#### 2) 실제 코드 위치

- `ChatMessageHandler#handleChatMessage`, `handleFileMessage`, `createMessageResponse`
- `SessionService#validateSession`, `updateLastActivity`
- `RateLimitService#checkRateLimit`, `RateLimitMongoStore`
- `RoomActivityNotifier#notifyMessageStored`, `RecentMessageCounter`
- Repository: Session/RateLimit/User/Room/Message/File Repository
- Document: `Session`, `RateLimit`, `User`, `Room`, `Message`, `File`

#### 3) 현재 코드 실행 흐름

`chatMessage` → session find+save → rate limit find+save → sender find → room find → (파일이면 file find) → message save → (파일이면 응답용 file 재조회) → 최근 메시지 count → 방송 → AI 시작 가능 → session find+save 재실행.

#### 4) 정확히 어디가 문제인가?

성공한 텍스트 메시지는 코드상 대략 session 4회(검증 find/save + 마지막 find/save), rate limit 2회, user 1회, room 1회, message save 1회, recent count 1회로 **약 10회**의 MongoDB 작업이 가능하다. 파일 메시지는 file을 검증과 응답 생성에서 2회 조회하여 약 12회다. 신규 RateLimit 삽입 방식이나 Spring Data 내부 동작에 따라 실제 명령 수는 달라질 수 있으므로 프로파일러로 확인해야 한다. AI 멘션이면 외부 AI 호출과 완료 시 메시지 save도 추가된다.

#### 5) 데이터가 많아지면 어떻게 되는가?

**예시:** 초당 메시지 1,000건이면 애플리케이션 논리상 초당 약 10,000회 수준의 Mongo 작업을 유발할 수 있다. 이는 측정치가 아니라 코드 경로 단순 합산 예시이며, 실제 드라이버 명령 수는 계측해야 한다.

#### 6) 서버에는 어떤 영향을 주는가?

MongoDB 부하·네트워크 증가, RPS 한계, p95/p99 증가, Socket.IO worker 지연, CPU/GC 증가, 연결 ping timeout 가능성이 생긴다.

#### 7) 부하테스트에서 어떻게 나타날까?

활성 사용자와 msg/s 증가 → MongoDB operation이 더 큰 배수로 증가 → DB CPU/latency 상승 → echo RTT p95/p99 증가 → Socket.IO event loop/worker 적체 → disconnect와 error rate 증가.

#### 8) 어떻게 부하테스트해야 하는가?

- 대상: Socket.IO `chatMessage`; text와 file을 분리
- 데이터: 사용자/방 사전 생성, AI mention 없는 고정 메시지부터 시작
- 부하: 50 → 100 → 250 → 500 VU, 사용자당 1/3/5 msg/s, 단계당 5분
- 확인: echo RTT p50/p95/p99, msg/s, error/disconnect, MongoDB collection별 operation 수·latency, CPU/memory/GC, Socket.IO worker queue 추정 지표
- `ramp-up-test.js`가 가장 직접적으로 검증 가능하며 echo RTT와 Mongo/Prometheus를 함께 봐야 한다. 개별 DB 단계 분해에는 tracing/profiler가 추가로 필요하다.

#### 9) 개선 방법

1순위: 세션 검증에서 한 번만 원자적으로 lastActivity를 갱신하고 마지막 중복 갱신을 제거한다. Rate Limit도 원자 increment로 통합한다. 2순위: 연결 시 검증된 사용자/방 멤버십을 안전한 짧은 캐시로 사용하고 무효화 정책을 둔다. 3순위: recent count를 매 메시지마다 query하지 말고 원자 카운터/시간 버킷/비동기 집계로 전환하며 파일 조회 결과를 재사용한다.

#### 10) 개선 전 / 개선 후 예상 구조

```text
개선 전: session 2쌍 → rate limit 1쌍 → user → room → message → count → 방송
개선 후: atomic session/rate limit → 캐시된 인증·멤버십 → message save → 비동기/원자 집계 → 방송
```

#### 11) 개선 효과를 어떻게 검증할까?

같은 msg/s와 VU에서 Before/After의 메시지당 Mongo operation, echo RTT p95/p99, 최대 지속 msg/s, DB CPU, disconnect를 비교한다. 최적화 때문에 권한·세션 만료·rate limit 정확성이 깨지지 않는지도 함께 검증한다.

#### 12) 핵심 정리

> 핵심: 텍스트 한 건이 약 10회의 MongoDB 작업으로 증폭될 수 있다.  
> 중복 세션 갱신, 비원자 Rate Limit, 매번 count를 먼저 줄여야 한다.

---

### 2.7 Rate Limit 원자성 문제

#### 1) 이 문제가 무엇인가?

Rate Limit은 일정 시간 동안 요청 수를 제한하는 기능이다. 카운터 증가가 원자적이지 않으면 동시 요청 여러 개가 같은 count를 읽고 모두 같은 다음 값으로 저장해 요청 일부가 카운트되지 않는다.

#### 2) 실제 코드 위치

- `RateLimitService#checkRateLimit`
- `RateLimitMongoStore#findByClientId`, `save`
- `RateLimitRepository#findByClientId`
- Document: `RateLimit`의 unique `clientId`, TTL `expiresAt`
- 호출: `RateLimitInterceptor#preHandle`, `ChatMessageHandler#handleChatMessage`

```java
RateLimit rateLimit = rateLimitStore.findByClientId(id).orElse(null);
int currentCount = rateLimit != null ? rateLimit.getCount() : 0;
rateLimit.setCount(currentCount + 1);
rateLimitStore.save(rateLimit);
```

#### 3) 현재 코드 실행 흐름

요청 A/B가 같은 clientId 조회 → 둘 다 count 10 확인 → 둘 다 허용 → 각각 11로 저장 → 실제 2요청 중 카운터는 1만 증가. 만료 window 재설정도 같은 경쟁을 겪는다.

#### 4) 정확히 어디가 문제인가?

`@Transactional`은 Java 메서드를 mutex로 만들지 않는다. Mongo 트랜잭션이 실제 활성화되려면 replica set과 transaction manager가 필요하지만 `MongoConfig`에는 manager가 보이지 않으며, 활성화되어도 동시에 읽은 카운터에 조건부 버전 검사 없이 Lost Update를 자동 방지하지 않는다. 신규 문서 동시 삽입은 unique 충돌이 날 수 있고 catch가 fail-open으로 허용한다. 또한 clientId 앞에 hostname을 붙여 노드별 제한이 되어 scale-out 시 전역 제한이 아니다.

#### 5) 데이터가 많아지면 어떻게 되는가?

**예시:** 제한 직전 동일 사용자 100개 동시 요청은 여러 요청을 허용하면서 count 증가가 일부만 남을 수 있다. 노드 3대라면 사용자가 노드별 키를 가져 사실상 제한이 분산될 수 있다. 정확한 초과 허용 수는 경쟁 타이밍에 따라 달라진다.

#### 6) 서버에는 어떤 영향을 주는가?

Race Condition, Lost Update, 제한 우회, 과도한 요청에 따른 MongoDB/CPU 부하, 노드별 정책 불일치가 발생한다.

#### 7) 부하테스트에서 어떻게 나타날까?

동일 identity 동시 burst → 허용 응답 수가 maxRequests 초과 → DB count와 실제 허용 수 불일치 → unique 오류 또는 fail-open 로그 → 이후 핵심 API 과부하.

#### 8) 어떻게 부하테스트해야 하는가?

- 대상: `@RateLimit`이 붙은 `GET /api/rooms`, Socket.IO `chatMessage`
- 데이터: 동일 사용자/토큰과 서로 다른 사용자 대조군
- 부하: window 시작에 10/100/500 동시 barrier burst; 단일 노드와 다중 노드 각각 100회 반복
- 확인: 허용/429 수, 최종 DB count, 제한 초과 허용, unique 오류, p95/p99
- 기존 스크립트는 사용자별로 요청하여 같은 키 경쟁 검증에 부적합하다. 동일 사용자 burst 전용 테스트가 필요하다.

#### 9) 개선 방법

1순위: MongoDB `findAndModify`의 조건부 `$inc` + upsert 또는 Redis Lua script로 “window 확인·증가·허용 판정”을 한 원자 연산으로 만든다. 2순위: hostname을 정책 키에서 제거하고 공유 저장소 기준 전역 제한을 적용한다. 3순위: 저장소 장애 시 fail-open/fail-closed 정책을 API 위험도별로 명시하고 metrics/alert를 둔다.

#### 10) 개선 전 / 개선 후 예상 구조

```text
개선 전: find → Java 판정/증가 → save → 경쟁 시 유실
개선 후: 공유 저장소 atomic increment+expiry+판정 1회 → 결과 반환
```

#### 11) 개선 효과를 어떻게 검증할까?

동일 window·limit·burst에서 Before/After의 실제 허용 수와 저장 count를 비교한다. 단일/다중 인스턴스 모두 정확히 제한되는지, window 경계와 저장소 장애 정책도 검증한다.

#### 12) 핵심 정리

> 핵심: Rate Limit 카운터가 find 후 save라 동시 요청이 유실된다.  
> 공유 저장소의 원자 increment/판정으로 바꾸고 hostname 기반 분할도 제거해야 한다.

---

### 2.8 방 참가 처리 비효율

#### 1) 이 문제가 무엇인가?

방 참가자 한 명이 바뀔 때 방의 모든 참가자 정보를 다시 읽고 전체 목록을 방 전체에 보내면 참가자 수에 비례해 DB와 네트워크 비용이 커진다. 이 항목은 경로별 구현이 다르다.

#### 2) 실제 코드 위치

- REST: `RoomController#joinRoom`, `RoomService#joinRoom`, 두 클래스의 `mapToRoomResponse`
- Socket.IO: `RoomJoinHandler#handleJoinRoom`, `RoomLeaveHandler#handleLeaveRoom`, `broadcastParticipantList`
- Repository: `RoomRepository#save`, `addParticipant`(`$addToSet`), `removeParticipant`(`$pull`), `UserRepository#findById`
- Document: `Room.participantIds`

```java
// REST: 전체 Room read-modify-save
room.getParticipantIds().add(user.getId());
roomRepository.save(room);

// Socket.IO: 이 부분은 이미 원자적
@Update("{'$addToSet': {'participantIds': ?1}}")
void addParticipant(String roomId, String userId);
```

#### 3) 현재 코드 실행 흐름

REST: room 조회 → user 조회 → 참가자 Set 변경 → 방 전체 save → 응답/이벤트용 creator·참가자 전원 조회 + count가 반복된다.  
Socket.IO: user 확인 → room 확인 → `$addToSet` → system message 저장 → 초기 메시지 조회·읽음 저장 → room 재조회 → 참가자 전원 개별 조회 → 입장 성공 → system message와 참가자 전체 목록 방송.

#### 4) 정확히 어디가 문제인가?

REST는 단일 참가자 추가에 방 문서 전체를 저장하며 동시 참가 시 Lost Update 가능성이 있다. 더구나 service와 controller에서 DTO 매핑을 각각 수행하여 참가 성공 시 생성자/참가자/count 조회가 두 번 반복된다. Socket.IO의 참가자 변경 자체는 `$addToSet`으로 원자적이므로 “전체 저장” 문제는 **현재 코드에서 확인되지 않는다**. 그러나 참가자 `P`명을 `P`회 조회하고 크기 `O(P)` 목록을 `P`명에게 방송하면 총 전달 데이터가 `O(P²)`에 가까워진다. 입장 시 초기 메시지 조회가 2번 발생한다(`RoomJoinHandler` 내부 load 후 클라이언트가 다시 fetch).

#### 5) 데이터가 많아지면 어떻게 되는가?

**예시:** 10명 방에서는 참가자 전체 JSON이 작지만 1,000명 방에 새 사용자 100명이 연속 입장하면 매번 1,000명 수준의 사용자 조회와 전체 방송이 반복된다. 실제 네트워크 bytes는 DTO 크기와 연결 수에 따라 다르다.

#### 6) 서버에는 어떤 영향을 주는가?

MongoDB operation 증가, 응답/입장 TTI 증가, CPU·메모리·GC, Socket.IO 네트워크와 worker/event loop 부하, REST 동시 참가의 Lost Update 가능성이 있다.

#### 7) 부하테스트에서 어떻게 나타날까?

동시 입장 증가 → 참가자 전원 조회 및 초기 메시지 읽음 저장 증가 → `participantsUpdate` fan-out 증가 → onboarding p95/p99 상승 → 네트워크 대역폭·MongoDB operation 급증 → 연결 실패/timeout.

#### 8) 어떻게 부하테스트해야 하는가?

- 대상: `POST /api/rooms/{id}/join`, Socket.IO `joinRoom`/`leaveRoom`
- 데이터: 방별 기존 참가자 10/100/1,000명, 메시지 30개 이상
- 부하: 신규 10 → 50 → 200 users/s, 5분; 같은 user 재입장 대조
- 확인: onboarding p50/p95/p99, REST latency, participant query 수, broadcast event/bytes, CPU/memory/GC, 최종 참가자 고유 수
- `ramp-up-test.js`가 REST+Socket join과 참가자 방송을 모두 수행해 직접 검증 가능하다. 큰 단일 방 집중 및 최종 정합성은 별도 모드가 필요하다.

#### 9) 개선 방법

1순위: REST도 `$addToSet` 원자 update 후 필요한 projection만 반환하고 DTO 매핑 중복을 제거한다. 2순위: 참가자 ID를 batch 조회하며 입장 응답의 초기 메시지와 클라이언트의 즉시 fetch 중 하나를 제거한다. 3순위: 전체 목록 대신 join/leave delta를 방송하고 클라이언트가 상태를 병합하거나 참가자 페이지 조회를 제공한다.

#### 10) 개선 전 / 개선 후 예상 구조

```text
개선 전: 1명 참가 → 전체 Room 저장/재조회 → 사용자 P회 조회 → 전체 P명 목록을 P명에게 방송
개선 후: atomic add → 신규 참가자 요약 1회 조회 → join delta 방송 → 필요 시 참가자 페이지 조회
```

#### 11) 개선 효과를 어떻게 검증할까?

같은 방 크기와 users/s로 Before/After를 실행한다. onboarding p95/p99, 참가 1건당 DB operation, participant broadcast bytes, 최종 참가자 정합성, duplicate system message/fetch 수를 비교한다.

#### 12) 핵심 정리

> 핵심: Socket.IO 참가 갱신은 이미 `$addToSet`이지만, 참가자 전원 조회·전체 방송은 남아 있다.  
> REST 전체 저장·중복 매핑과 전체 목록 fan-out을 함께 줄여야 한다.

---

### 2.9 추가 인덱스 부재

#### 1) 이 문제가 무엇인가?

Collection Scan은 조건에 맞는 문서를 찾기 위해 컬렉션의 많은 문서를 직접 검사하는 실행 계획이다. 이 프로젝트의 파일 권한 검사와 최근 메시지 count에는 `_id`가 아닌 필드 조회가 있지만 코드에 대응 인덱스가 선언되지 않았다.

#### 2) 실제 코드 위치

- `MessageRepository#countRecentMessagesByRoomId`: `room + timestamp`
- `MessageRepository#findByFileId`: `Message.file`
- `FileRepository#findByFilename`: `File.filename`
- 호출: `RecentMessageCounter`, `FileAccessService#authorize`
- Document: `Message`, `File`; 두 클래스에 `@Indexed`/`@CompoundIndex` 없음

```java
@Query(value = "{ 'room': ?0, 'timestamp': { $gte: ?1 } }", count = true)
long countRecentMessagesByRoomId(...);
Optional<Message> findByFileId(String fileId);
Optional<File> findByFilename(String filename);
```

#### 3) 현재 코드 실행 흐름

방 목록/메시지 전송 → 최근 30분 room count.  
파일 download/view → filename으로 file 조회 → fileId로 message 조회 → `_id`로 room 조회 → 권한 확인.

#### 4) 정확히 어디가 문제인가?

`User.email`, RateLimit, Session에는 인덱스 선언이 있지만 Message/File 조회 필드에는 없다. 운영 DB에 수동 인덱스가 없다면 `files.filename`, `messages.file`, `messages.room+timestamp`에서 COLLSCAN 가능성이 있다. `Optional` 반환은 첫 결과만 필요해도 인덱스 부재 비용을 없애지 못한다. 실제 실행 계획은 DB에서만 확정할 수 있다.

#### 5) 데이터가 많아지면 어떻게 되는가?

**예시:** 파일/메시지 각각 100건이면 눈에 띄지 않지만 각 1,000,000건에서 특정 filename/fileId 한 건을 찾을 때 검사 문서 수가 크게 증가할 수 있다. 메시지 전송마다 recent count가 실행되어 이 비용이 빈도와 곱해진다.

#### 6) 서버에는 어떤 영향을 주는가?

MongoDB CPU·디스크 I/O·메모리 증가, 응답 p95/p99 증가, RPS 감소, 메시지 echo와 파일 응답 지연이 발생한다.

#### 7) 부하테스트에서 어떻게 나타날까?

컬렉션 크기 증가 → 같은 RPS에서도 `docsExamined`와 DB CPU 증가 → count/파일 권한 query 지연 → 메시지 및 파일 API p95/p99 증가 → 처리량 정체.

#### 8) 어떻게 부하테스트해야 하는가?

- 대상: `chatMessage`(recent count), `/api/files/download/{filename}`, `/view/{filename}`
- 데이터: messages/files 1만/10만/100만 단계, 존재/부재 filename과 여러 room 분포
- 부하: 각 API 1/10/50 VU, 단계당 3분
- 확인: `explain("executionStats")`, winningPlan, `docsExamined`, `keysExamined`, p50/p95/p99, RPS, DB CPU/I/O
- `ramp-up-test.js`는 메시지 count와 업로드만 포함하고 download/view는 없다. 인덱스별 격리 테스트가 필요하다.

#### 9) 개선 방법

1순위: 실제 cardinality와 쿼리로 검증 후 `files.filename`(보통 unique 여부 검토), `messages.file`(필요 시 sparse/partial), `messages.room + timestamp` 인덱스를 마이그레이션으로 관리한다. 2순위: 파일 권한 경로 projection으로 필요한 필드만 읽는다. 3순위: 사용되지 않거나 중복된 인덱스는 쓰기 비용을 늘리므로 운영 쿼리 통계로 정리한다.

#### 10) 개선 전 / 개선 후 예상 구조

```text
개선 전: collection 문서 검사 → 조건 일치 문서 발견
개선 후: index key 탐색(IXSCAN) → 소수 문서 FETCH → 응답
```

#### 11) 개선 효과를 어떻게 검증할까?

동일 데이터·쿼리에서 Before/After explain을 저장한다. `COLLSCAN/IXSCAN`, examined/returned 비율, execution time, p95/p99, DB CPU/I/O와 인덱스 추가 후 쓰기 latency·저장 용량을 비교한다.

#### 12) 핵심 정리

> 핵심: filename, fileId, room+timestamp 조회를 지원하는 코드 선언 인덱스가 없다.  
> 운영 `getIndexes`와 `explain`으로 확인하고 쿼리별 최소 인덱스를 관리해야 한다.

---

### 2.10 파일 업로드/다운로드 서버 점유

#### 1) 이 문제가 무엇인가?

Blocking I/O는 디스크나 네트워크 작업이 끝날 때까지 처리 스레드가 기다리는 방식이다. 기본 설정은 로컬 디스크이고 Spring MVC/Tomcat 요청에서 업로드 파일을 동기 복사하며, 다운로드/보기도 서버가 `Resource`를 응답한다.

#### 2) 실제 코드 위치

- `FileController#uploadFile`, `downloadFile`, `viewFile`
- `LocalFileService#uploadFile`
- `FileAccessService#authorize`, `issue`
- `LocalStorage#put`, `open`
- Repository: User/File/Message/Room Repository
- 설정: `file.storage.type=local`, `file.upload-dir=./uploads`, multipart 5MB, `server.tomcat.threads.max=10`

```java
Files.copy(content, targetPath, StandardCopyOption.REPLACE_EXISTING);
// offload URL이 없으면
return new FileAccess.Stream(resource, ..., fileEntity.getSize());
```

#### 3) 현재 코드 실행 흐름

업로드: multipart 수신 → user 조회 → 파일 검증 → 요청 스레드에서 로컬 디스크 copy → File metadata save → 응답.  
다운로드/보기: user 조회 → filename/fileId/room 3홉 권한 조회 → 로컬 `UrlResource` 열기 → Tomcat 응답으로 파일 bytes 전달.

#### 4) 정확히 어디가 문제인가?

기본 `LocalStorage#offloadUrl`은 인터페이스 기본 구현상 비어 있으므로 서버 경유 스트리밍이다. 업로드의 `Files.copy`는 동기 호출이다. 다운로드는 Spring이 resource를 쓰는 동안 servlet 요청이 완료되지 않으므로 느린 클라이언트·큰 파일이 동시 발생하면 제한된 Tomcat thread/connection을 오래 점유할 수 있다. 현재 제한은 5MB라 문서 주석의 50MB와 다르지만, 5MB도 동시 10개에서 영향을 줄 수 있다.

#### 5) 데이터가 많아지면 어떻게 되는가?

**예시:** 1개 100KB 업로드와 달리 5MB 업·다운로드가 동시에 10개 지속되면 최대 10개 Tomcat worker가 파일 작업에 묶일 수 있다. 디스크 처리량과 클라이언트 속도에 따라 일반 REST 요청도 대기할 수 있으며 정확한 시간은 측정해야 한다.

#### 6) 서버에는 어떤 영향을 주는가?

Tomcat thread 점유, 디스크 I/O, 네트워크 사용량, 메모리/버퍼 사용, 응답시간 증가, RPS와 accept 여유 감소가 발생한다.

#### 7) 부하테스트에서 어떻게 나타날까?

동시 업/다운로드 증가 → active Tomcat threads 10에 접근 → request queue 증가 → 파일 p95/p99뿐 아니라 `/api/health` 등 일반 API p95도 상승 → timeout/error → 디스크 util·네트워크 포화.

#### 8) 어떻게 부하테스트해야 하는가?

- 대상: `/api/files/upload`, `/download/{filename}`, `/view/{filename}`와 일반 API 대조군
- 데이터: 권한이 연결된 100KB/1MB/5MB 파일; 빠른/속도 제한 다운로드 클라이언트
- 부하: 파일 VU 1 → 5 → 10 → 20, 각 5분; 동시에 일반 API 5 VU 유지
- 확인: 파일/일반 API p50/p95/p99, RPS, active/busy Tomcat threads, queue, CPU/memory/GC, disk latency/util, network throughput
- `ramp-up-test.js`는 17번째 메시지마다 작은 이미지 업로드만 한다. 크기별 업로드와 download/view, 느린 소비자는 별도 테스트가 필요하다.

#### 9) 개선 방법

1순위: S3 같은 object storage에 presigned upload/download 또는 X-Accel-Redirect/X-Sendfile 방식으로 data plane을 앱에서 분리한다. 2순위: 로컬 유지 시 파일 전용 인스턴스/스레드·동시성 제한, 적절한 timeout과 스트리밍 버퍼를 둔다. 3순위: multipart 크기·확장자·속도 제한과 저장 실패 시 고아 파일 정리, 파일 메타데이터 인덱스를 적용한다.

#### 10) 개선 전 / 개선 후 예상 구조

```text
개선 전: Client ↔ Tomcat 인증/DB 조회 ↔ 로컬 디스크 bytes 전송
개선 후: Client → 앱에서 권한 확인/짧은 URL 발급 → object storage/CDN과 직접 전송
```

#### 11) 개선 효과를 어떻게 검증할까?

같은 파일 크기·동시성·네트워크 제한에서 Before/After를 실행한다. 앱 outbound bytes, busy thread, disk I/O, 일반 API p95/p99, 파일 성공률을 비교하고 URL 만료와 권한 우회가 없는지도 검증한다.

#### 12) 핵심 정리

> 핵심: 기본 로컬 저장소는 업로드를 동기 복사하고 다운로드 bytes도 앱 서버가 전달한다.  
> 최대 10개 Tomcat thread 환경에서는 파일 I/O를 외부 저장소로 오프로딩하는 효과가 크다.

---

### 2.11 AI 스트리밍 누적 전송

#### 1) 이 문제가 무엇인가?

Streaming은 결과가 완성되기 전에 작은 chunk로 전달하는 방식이다. 그런데 이 코드는 새 chunk만 보내지 않고 지금까지 누적된 전체 문자열을 매번 다시 보낸다. 결과 길이가 늘수록 중복 전송량이 커진다.

#### 2) 실제 코드 위치

- `AiService#startStreaming`, `streamResponse`, `onAiMessageCompleteEvent`
- `AiStreamHandler#onSubscribe`, `onNext`
- `StreamingSession#appendContent`
- `SocketIOEventListener#handleAiMessageChunkEvent`
- 이벤트: `AiMessageChunkEvent.fullContent`, Socket.IO `aiMessageChunk`

```java
subscription.request(Long.MAX_VALUE);
session.appendContent(chunk.currentChunk());
publishEvent(new AiMessageChunkEvent(..., session.getContent(), ...));
// listener가 fullContent를 방 전체 전송
```

#### 3) 현재 코드 실행 흐름

AI mention → OpenAI content Flux 구독 → chunk마다 `content += chunk` → 누적 `session.getContent()` 이벤트 발행 → 방 전체에 `fullContent` 전송 → 완료 시 전체 내용으로 Message save 및 complete 방송.

#### 4) 정확히 어디가 문제인가?

길이가 `c`인 chunk가 `n`개면 전송 문자열 총량은 대략 `c(1+2+...+n)`, 즉 `O(n²)`이다. Java의 immutable String에 `content +=`를 사용하여 누적 복사와 임시 객체도 반복된다. `request(Long.MAX_VALUE)`로 upstream backpressure를 사실상 해제하고, 느린 Socket.IO 소비자에 맞춘 chunk 병합/throttle도 없다. 각 chunk는 방의 모든 클라이언트에 fan-out된다.

#### 5) 데이터가 많아지면 어떻게 되는가?

**예시:** 같은 크기 chunk 100개라면 신규 chunk만 전송할 때 100단위지만 누적 전송은 5,050단위가 된다. 여기에 방 참가자 수가 곱해진다. 이는 상대량 예시이며 실제 bytes는 chunk와 JSON/프로토콜 overhead에 따라 다르다.

#### 6) 서버에는 어떤 영향을 주는가?

CPU 증가, 메모리와 GC 증가, Socket.IO 네트워크 사용량·event loop/worker 부하, 느린 클라이언트 큐 적체, 다른 메시지 latency 증가가 발생한다.

#### 7) 부하테스트에서 어떻게 나타날까?

동시 AI stream·응답 길이·방 인원 증가 → outbound bytes가 생성 token보다 빠르게 증가 → heap/GC와 CPU 상승 → 일반 `message` p95/p99 상승 → ping timeout/disconnect.

#### 8) 어떻게 부하테스트해야 하는가?

- 대상: AI mention이 포함된 `chatMessage`, `aiMessageChunk`
- 데이터: 100/1,000/5,000자 응답을 재현하는 stub AI가 권장됨; 방 참가자 1/10/100명
- 부하: 동시 stream 1 → 10 → 50, 각 단계 5분; 일부 느린 소비자 포함
- 확인: AI TTFB/완료 p50/p95/p99, chunk 수, 생성 문자 대비 outbound bytes, CPU/memory/GC, Socket.IO queue/latency, disconnect
- 기존 스크립트는 AI mention을 보내지 않아 검증 불가하다. 비용·변동성을 통제하는 stub 기반 별도 테스트가 필요하다.

#### 9) 개선 방법

1순위: 이벤트에는 `delta`와 순번만 보내고 클라이언트가 누적한다. 2순위: `StringBuilder` 또는 적절한 buffer를 사용하고 20~100ms 단위로 chunk를 병합/throttle한다. 3순위: 무제한 demand 대신 backpressure/queue 상한, 느린 소비자 정책, 최대 출력 길이와 동시 stream 제한을 둔다.

#### 10) 개선 전 / 개선 후 예상 구조

```text
개선 전: chunk → 전체 문자열 재생성 → fullContent를 매번 방 전체 전송
개선 후: chunk buffer → 주기적 delta+sequence 전송 → client 누적 → 완료 시 최종 저장
```

#### 11) 개선 효과를 어떻게 검증할까?

동일 stub 응답과 방 인원으로 Before/After를 비교한다. 생성 문자당 outbound bytes, heap allocation/GC, CPU, chunk event 수, AI 완료 시간과 일반 채팅 p95/p99를 측정한다. chunk 누락·순서 역전 시 재조립 정책도 검증한다.

#### 12) 핵심 정리

> 핵심: AI chunk마다 누적 전체 문자열을 다시 보내 전송량과 문자열 복사가 제곱 수준으로 커진다.  
> delta 전송, chunk 병합, backpressure가 필요하다.

---

### 2.12 Socket.IO 확장성 문제

#### 1) 이 문제가 무엇인가?

Scale-out은 서버 인스턴스를 여러 대로 늘려 부하를 나누는 것이다. 연결·룸 상태가 각 JVM 메모리에만 있으면 다른 노드가 그 상태를 모르므로 방송과 재연결이 일관되게 동작하지 않는다. backlog는 OS가 아직 accept되지 않은 연결을 기다리게 하는 큐 크기다.

#### 2) 실제 코드 위치

- `SocketIOConfig#socketIOServer`: `setAcceptBackLog(10)`, `MemoryStoreFactory`
- `SocketIOConfig#chatDataStore`: `new LocalChatDataStore()`
- `LocalChatDataStore`: `ConcurrentHashMap`
- 사용: `ConnectedUsers`, `UserRooms`
- 설정: Socket.IO port 5002; REST Tomcat accept-count도 10이지만 이 이슈의 직접 대상은 Socket.IO Netty 설정

```java
socketConfig.setAcceptBackLog(10);
config.setStoreFactory(new MemoryStoreFactory());
return new LocalChatDataStore();
```

#### 3) 현재 코드 실행 흐름

클라이언트 연결 → 특정 인스턴스의 Netty 서버가 accept → socket room과 Socket.IO store가 해당 JVM 메모리에 저장 → `UserRooms`/`ConnectedUsers`도 로컬 Map 사용 → 방송은 해당 인스턴스가 아는 room client에게만 전달.

#### 4) 정확히 어디가 문제인가?

연결 burst가 10개 대기열을 빠르게 넘으면 OS/환경에 따라 연결 거부·timeout 가능성이 커진다. `MemoryStoreFactory`와 로컬 Map은 노드 간 공유되지 않는다. 로드밸런서에 sticky session이 있어도 다른 노드에서 발생한 room event를 pub/sub로 전달할 수 없으며, 재연결 노드가 바뀌면 `UserRooms` 상태가 사라진다. `ConcurrentHashMap`은 map 자체에는 안전하지만 `UserRooms#get → 복사 → set`은 동일 사용자의 동시 add/remove에 원자적이지 않다.

#### 5) 데이터가 많아지면 어떻게 되는가?

**예시:** 평상시 초당 몇 연결은 통과하지만 배포·네트워크 복구 후 수백 사용자가 동시에 재연결하면 backlog가 작아 실패가 집중될 수 있다. 노드 2대로 늘리면 같은 방 사용자가 양쪽에 나뉘어 한 노드의 메모리 방송만으로는 전체 전달이 보장되지 않는다.

#### 6) 서버에는 어떤 영향을 주는가?

연결 오류·ping timeout, 메시지/participant 방송 누락, 노드별 상태 불일치, Socket.IO event loop/worker 부하, scale-out 불가 또는 sticky session 강제, 메모리 증가가 발생한다.

#### 7) 부하테스트에서 어떻게 나타날까?

연결 ramp/burst 증가 → connect p95/p99와 connection error 상승 → backlog 포화 → disconnect 증가. 다중 노드에서는 같은 room의 일부 클라이언트만 event 수신, 재연결 후 중복 join/system message 또는 상태 누락이 나타날 수 있다.

#### 8) 어떻게 부하테스트해야 하는가?

- 대상: Socket.IO connect/auth, `joinRoom`, `chatMessage`, broadcast
- 데이터: 같은 방/여러 방, 사전 인증 토큰
- 부하: ramp 10 → 100 users/s 및 500/1,000 동시 burst, 10분 sustain; 단일 노드와 2노드 비교
- 확인: connect p50/p95/p99, connection error, active connections, ping timeout, disconnect reason, event 전달 완전성, 노드별 CPU/memory/network
- `ramp-up-test.js`는 연결 ramp, p95/p99, ping timeout, server disconnect를 측정해 단일 노드 한계 확인에 적합하다. burst와 다중 노드 cross-node 전달은 별도 테스트가 필요하다.

#### 9) 개선 방법

1순위: 실제 트래픽과 OS 한계에 맞춰 backlog, worker 수, buffer를 측정 기반 조정하고 로드밸런서 연결 timeout/sticky 정책을 정렬한다. 2순위: netty-socketio의 Redis/Redisson store·pub/sub 지원을 검증해 room/session broadcast를 공유한다. 3순위: `UserRooms`를 공유 저장소의 원자 set 연산으로 바꾸고 재연결·노드 장애·중복 이벤트 처리 규칙을 둔다.

#### 10) 개선 전 / 개선 후 예상 구조

```text
개선 전: LB → Node A 메모리 room / Node B 메모리 room (서로 모름)
개선 후: LB(sticky 필요성 검토) → 여러 Socket 노드 ↔ 공유 store/pub-sub → 전체 room 전달
```

#### 11) 개선 효과를 어떻게 검증할까?

동일 연결 ramp/burst와 메시지 패턴으로 Before/After를 실행한다. 연결 성공률, connect p95/p99, ping timeout, 모든 구독자의 event 수신율, 노드별 부하 분산을 비교하고 한 노드 강제 종료 후 재연결·상태 복구도 검증한다.

#### 12) 핵심 정리

> 핵심: backlog 10과 JVM 인메모리 room/user 상태는 연결 burst와 다중 노드 확장을 막는다.  
> 공유 store/pub-sub와 원자 상태 관리, 측정 기반 네트워크 튜닝이 필요하다.

---

## 3. 추가 개념 설명

| 용어 | 설명 | 이 프로젝트의 예 |
|---|---|---|
| N+1 | 목록 쿼리 1번 뒤 항목마다 추가 쿼리를 반복하는 패턴 | 방마다 사용자/count, 메시지마다 사용자/파일 조회 |
| 복합 인덱스 | 여러 필드를 순서 있게 묶은 인덱스 | `{ room: 1, timestamp: -1 }` 후보 |
| Collection Scan | 컬렉션 문서를 넓게 직접 검사하는 실행 계획(`COLLSCAN`) | 인덱스가 없다면 filename/fileId/room 조회에서 가능 |
| Index Scan | 인덱스 키로 후보를 좁히는 실행 계획(`IXSCAN`) | `_id` 조회와 적절한 추가 인덱스 조회 |
| read-modify-save | 문서를 읽고 애플리케이션에서 바꾼 뒤 다시 저장 | readers/reactions, REST participant 변경 |
| Race Condition | 실행 순서에 따라 결과가 달라지는 동시성 결함 | 동일 메시지에 동시 읽음·반응 |
| Lost Update | 나중 저장이 다른 요청의 변경을 덮어 변경이 사라지는 현상 | 두 사용자의 reaction 중 하나가 소실될 수 있음 |
| 원자적 연산(Atomic Operation) | 다른 작업이 중간 상태를 볼 수 없는 하나의 불가분 DB 연산 | `$addToSet`, `$pull`, `$inc`, 조건부 `findAndModify` |
| Rate Limit | 사용자/IP가 시간 창 안에 보낼 수 있는 요청 수 제한 | REST interceptor와 `chatMessage`의 Mongo 카운터 |
| DB Round Trip | 앱이 DB에 요청하고 결과를 받는 한 번의 네트워크 왕복 | `findById` 한 번, `save` 한 번이 각각 왕복 |
| Blocking I/O | I/O가 끝날 때까지 실행 스레드가 기다리는 방식 | `Files.copy`, 서버 경유 파일 응답 |
| Streaming | 전체 결과 전 작은 조각을 연속 전달하는 방식 | Spring AI `Flux<String>`과 `aiMessageChunk` |
| Backpressure | 소비 속도에 맞춰 생산 속도·버퍼를 제어하는 메커니즘 | 현재 AI 구독은 `Long.MAX_VALUE` 요청이라 실질적 제어가 약함 |
| Socket.IO | 실시간 양방향 이벤트 통신 라이브러리/프로토콜 | `joinRoom`, `chatMessage`, `messageReaction` 이벤트 |
| Scale-out | 서버 사양을 키우기보다 인스턴스 수를 늘리는 확장 | 여러 Socket.IO 노드가 공유 store/pub/sub를 사용하는 구조 |
| p50 / p95 / p99 | 지연 표본의 50/95/99 백분위. p99는 가장 느린 1% 경계 | 평균만으로 숨겨지는 tail latency 확인 |
| RPS | 초당 완료된 요청 수(Requests Per Second) | REST 처리량. Socket 이벤트는 events/s 또는 msg/s로 별도 표기 권장 |
| VU | 부하 도구가 모사하는 가상 사용자(Virtual User) | 동시 접속·행동을 수행하는 테스트 사용자 |

### 인덱스 결과 해석 시 주의점

- `keysExamined = 0`이 항상 좋은 것은 아니다. `COLLSCAN`은 인덱스 키를 보지 않아 0일 수 있으므로 winning plan과 `docsExamined`를 함께 본다.
- 좋은 point/range query는 일반적으로 반환 문서 수에 가까운 `docsExamined`를 기대하지만, 정확한 기준은 데이터 분포와 쿼리에 따라 정한다.
- 인덱스는 읽기를 빠르게 하는 대신 모든 insert/update의 쓰기 비용과 디스크 사용량을 늘린다. “필드마다 인덱스”가 아니라 실제 쿼리 조합을 기준으로 선택한다.

---

## 4. 부하테스트 우선순위 TOP 5

| 순위 | 문제 | 왜 우선순위가 높은가? | 현재 스크립트로 검증 가능한가? | 별도 테스트 |
|---:|---|---|---|---|
| 1 | 6. 메시지 전송당 DB 접근 과다 | 핵심 채팅 hot path이며 메시지마다 약 10회 Mongo 작업 가능. 4·7·9번 비용도 한 경로에 모인다. | **`ramp-up-test.js`로 직접 가능**. 지속 msg/s, echo p95/p99, disconnect를 수집한다. `load-test.js`의 latency는 emit 직후 값을 기록하는 구간이 있어 주 지표로 부적절하다. | Mongo profiler/APM으로 메시지당 collection별 operation을 분해해야 한다. |
| 2 | 4. 읽음 처리 반복 DB 작업 | 조회 30개가 최대 60개 DB 작업으로 증폭되고 기존 ramp가 메시지마다 읽음도 보낸다. | **둘 다 가능**, 특히 ramp에서 fetch와 실시간 1개 읽음이 함께 발생한다. | 1/30/100 IDs와 이미 읽음/no-op을 통제한 batch 테스트 필요 |
| 3 | 1. 방 목록 전체 조회 + N+1 | 데이터가 늘수록 쿼리 수·응답 크기가 무제한 증가하며 REST thread를 오래 점유한다. | **`ramp-up-test.js`에서 부분 가능**: 매 방 생성 전에 목록 조회 | 방/참가자 수를 미리 고정하고 `GET /api/rooms`만 반복하는 HTTP 테스트 필요 |
| 4 | 12. Socket.IO 확장성 | backlog 10은 연결 ramp의 조기 병목이고 인메모리 구조는 scale-out 정확성 문제다. | **`ramp-up-test.js`로 단일 노드 ramp 가능**: connect p95/p99, error, ping timeout, disconnect 제공 | 동시 burst, 2개 이상 노드, cross-node event 완전성, node kill 테스트 필요 |
| 5 | 2. 메시지 조회 복합 인덱스 부재 | 입장과 스크롤의 핵심 조회이며 데이터량에 따라 DB scan/sort 비용이 급증한다. | **부분 가능**: 두 스크립트 모두 첫 페이지 fetch | 대용량 seed, 깊은 cursor, `explain` 수집 전용 테스트 필요 |

동시성 정합성만 놓고 보면 5번과 7번도 최우선이다. 다만 위 TOP 5는 “현재 부하 스크립트를 활용해 성능 개선 프로젝트를 시작하기 좋은 순서”를 기준으로 선정했다. 5·7번은 일반 latency 테스트만으로 놓치기 쉬워 별도의 barrier 기반 정합성 테스트 트랙에서 병행해야 한다.

### 기존 스크립트 활용 요약

- `load-test.js`: 사용자 생성, Socket 연결, join, 이전 메시지 fetch, text 전송, 읽음, 10% reaction을 수행한다. 다만 메시지 latency 배열은 `emit` 직후 차이를 기록하므로 서버 왕복 latency로 해석하면 안 된다.
- `ramp-up-test.js`: REST join/room 조회, Socket join/fetch/chat/read/reaction, 주기적 파일 upload, ramp/sustain, echo RTT와 socket/REST p50·p95·p99, 오류·disconnect를 제공하므로 현재 종합 테스트의 우선 도구다.
- 두 스크립트 모두 파일 download/view, AI mention, 동일 ID 동시성 barrier, 다중 노드 전달 완전성은 직접 검증하지 않는다.

---

## 5. 추천 진행 순서

### 5.1 공통 반복 절차

```text
문제 발견
→ 실제 코드와 실행 경로 분석
→ 측정 가능한 성능 저하/정합성 가설 설정
→ 데이터·환경·부하 조건 고정
→ Before 부하테스트 및 DB explain/profiler 수집
→ 병목/오류/정합성 결과 분석
→ 한 번에 한 핵심 변경 적용
→ 동일 조건 After 부하테스트
→ p50/p95/p99·RPS·자원·DB operation·정합성 Before/After 비교
→ 회귀 테스트와 결론 기록
```

### 5.2 이 프로젝트에 권장하는 단계

1. **관측 기준선 확립**: 앱/노드 CPU, heap·GC, Tomcat busy thread, Socket 연결·disconnect, MongoDB collection별 operation/latency, network를 같은 시간축으로 수집한다.
2. **데이터셋 고정**: 방·참가자·메시지·파일 수와 분포를 seed manifest로 기록한다. Before와 After 사이에 데이터량이 달라지면 비교가 무효화된다.
3. **핵심 hot path 측정**: `ramp-up-test.js`로 6·4·12번의 breaking point를 찾되 AI와 파일 다운로드는 제외해 원인을 좁힌다.
4. **조회 격리 측정**: 방 목록(1), 메시지 cursor(2·3), 인덱스 후보(9)를 각각 별도 시나리오로 실행하고 Mongo `explain`을 저장한다.
5. **정합성 측정**: 5번은 동일 messageId, 7번은 동일 clientId에 barrier burst를 가해 성공 수와 최종 DB 상태를 비교한다. latency만 보고 통과시키지 않는다.
6. **I/O·AI·분산 측정**: 파일 크기/느린 소비자(10), stub AI/방 fan-out(11), 다중 노드와 node kill(12)을 독립 환경에서 검증한다.
7. **개선은 한 축씩 적용**: 예를 들어 메시지 전송에서 세션 중복 갱신 제거, Rate Limit 원자화, count 비동기화를 한꺼번에 적용하지 말고 각 변경의 효과를 구분한다.
8. **동일 조건 재실행**: VU, ramp, duration, 데이터, JVM 옵션, DB tier, 네트워크, warm-up을 동일하게 유지하고 최소 3회 반복해 중앙값과 변동 폭을 기록한다.
9. **성능과 정확성을 함께 판정**: p95가 낮아져도 누락된 reader/reaction, 초과 허용된 Rate Limit, 빠진 cross-node event가 있으면 개선으로 인정하지 않는다.

### 5.3 Before/After 비교 체크리스트

| 계층 | 비교 지표 |
|---|---|
| 사용자 체감 | REST/Socket p50·p95·p99, connect/TTI, AI TTFB·완료 시간, 파일 완료 시간 |
| 처리량/안정성 | RPS, msg/s, error rate, timeout, disconnect 및 ping timeout |
| 애플리케이션 | CPU, memory, allocation/GC, Tomcat active/busy threads, Socket 연결 수 |
| MongoDB | operation 수, query/update latency, `docsExamined`, `keysExamined`, plan, CPU/I/O, bytes read/written |
| 네트워크 | 응답 bytes, Socket outbound bytes, 파일 throughput, 생성 AI 문자 대비 전송 bytes |
| 정합성 | 최종 참가자·reader·reaction 고유 수, Rate Limit 허용 수, 모든 Socket 구독자의 event 수신율 |

> 최종 원칙: “코드상 좋아 보임”이 아니라, 동일 조건의 Before/After에서 병목 지표가 개선되고 기능·정합성이 유지될 때만 해결로 판정한다.
