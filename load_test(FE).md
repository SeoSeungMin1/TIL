# 프론트엔드 성능 취약점 검증 및 개선 가이드

이 문서는 현재 프론트엔드 소스 코드를 정적으로 다시 확인한 결과다. 이전에 제기된 7개 항목을 사실로 전제하지 않았으며, 실제 호출 경로와 정리(cleanup) 코드를 따라가며 판정했다.

> 주의: 이 문서의 복잡도와 영향은 코드 분석에 근거한 가설이다. 별도의 브라우저 프로파일링이나 부하 측정은 수행하지 않았으므로 시간, FPS, 메모리, RPS 등의 실측값은 포함하지 않는다. 백엔드 내부 구현은 이 프론트엔드 디렉터리만으로 확인할 수 없으며, 해당 부분은 반드시 별도 계측해야 한다.

---

## 1. 전체 요약

| 번호 | 문제 | 심각도 | 관련 기능 | 핵심 원인 | 실제 코드에서 확인 여부 |
| --- | --- | --- | --- | --- | --- |
| 1 | 방 목록 30초 Polling | 중간 | 네트워크/API 요청, React 렌더링, Socket.IO | 연결된 상태의 보이는 탭마다 30초 간격으로 전체 `GET /api/rooms` 실행 | 확인됨 |
| 2 | 방 입장 중복 요청 | 중간 | 네트워크/API 요청, Socket.IO | 목록에서 REST 가입 후 상세 REST 조회와 Socket.IO 참가를 수행하지만 목적이 서로 다르고, 메시지 추가 조회는 조건부임 | 부분 확인됨 |
| 3 | 방 설정 순차 실행 | 중간 | 네트워크/API 요청, Socket.IO | 소켓 연결 완료 후에야 독립 실행 가능한 방 상세 REST 조회를 시작함 | 확인됨 |
| 4 | 전체 메시지 계속 정렬/렌더링 | 높음 | JavaScript 연산, React 렌더링, DOM/브라우저 렌더링 | 병합 시 전체 정렬, 표시 시 다시 전체 정렬하며 누적 메시지를 모두 React 요소와 DOM으로 유지 | 확인됨 |
| 5 | 읽음 이벤트 배열 탐색 | 높음 | Socket.IO, JavaScript 연산, React 렌더링 | 읽음 이벤트마다 전체 메시지를 순회하고 `messageIds.includes`, 메시지별 참여자-읽은 사용자 중첩 탐색 수행 | 확인됨 |
| 6 | 입력마다 DOM/레이아웃 계산 | 중간 | DOM/브라우저 렌더링, React 렌더링 | 일반 입력은 state 갱신만 확인되나 활성 멘션 입력 중에는 매 키 입력마다 스타일·폭·위치 측정과 임시 DOM 삽입 수행 | 부분 확인됨 |
| 7 | 파일 Preview 메모리 관리 | 낮음 | 메모리 관리 | 주요 경로에 revoke와 파일 크기 제한이 있으나, 반복 선택 개수 제한과 URL 소유권이 분산되어 있음 | 부분 확인됨 |

분류별로 보면 네트워크/API 요청은 1~3번, Socket.IO는 1~3번과 5번, React 렌더링은 1번과 4~6번, JavaScript 연산은 4~5번, DOM/브라우저 렌더링은 4번과 6번, 메모리 관리는 4번과 7번에 해당한다.

---

## 2. 각 취약점 상세 분석

## 2.1 방 목록 30초 Polling

### 1) 이 문제가 무엇인가?

Polling은 클라이언트가 변경 여부와 관계없이 일정 간격으로 서버에 데이터를 다시 요청하는 방식이다. 이 프로젝트는 방 목록에 Socket.IO 변경 이벤트를 반영하면서도, 활성도 지표를 보정하기 위해 연결된 사용자의 보이는 탭에서 30초마다 전체 방 목록을 다시 가져온다.

한 사용자의 요청은 작아 보여도 사용자 수에 비례해 요청이 늘어난다. 예를 들어 동시에 목록을 보고 있는 사용자가 `U`명이라면 이론적인 평균 목록 요청률은 약 `U / 30` requests/sec다. 이는 코드에서 유도한 산식이지 실측값이 아니다. 응답이 전체 방과 참여자 정보를 포함한다면 전송량과 서버 조회 비용도 함께 커질 수 있다.

### 2) 실제 코드 위치

- `features/chat/rooms/ChatRoomsView.js:22`: `ROOM_LIST_REFRESH_INTERVAL = 30000`
- `features/chat/rooms/ChatRoomsView.js:69-102`: 최초 `fetchRooms()` 실행
- `features/chat/rooms/ChatRoomsView.js:120-135`: 30초 interval, visibility 검사, cleanup
- `features/chat/rooms/useRoomList.js:59-69`: `loadRooms()`, 연결 검사 후 `GET /api/rooms`
- `features/chat/rooms/useRoomList.js:99-127`: `refreshRooms()`
- `features/chat/rooms/useRoomsSocket.js`: `roomCreated`, `roomUpdated`, `roomActivity` 증분 반영

```javascript
const ROOM_LIST_REFRESH_INTERVAL = 30000;

const refreshWhenVisible = () => {
  if (document.visibilityState !== 'visible') return;
  refreshRoomsRef.current({ silent: true });
};

const refreshTimer = setInterval(
  refreshWhenVisible,
  ROOM_LIST_REFRESH_INTERVAL
);
```

### 3) 현재 코드 실행 흐름

방 목록 진입
→ `ChatRoomsView` mount
→ 최초 `fetchRooms()`
→ `loadRooms()`
→ `attemptConnection()`
→ `GET /api/rooms`
→ 전체 `rooms` state 교체
→ Socket.IO의 방 생성·수정·활성도 이벤트도 별도 반영
→ 연결 상태가 `CONNECTED`이고 탭이 보이면 30초마다 `refreshRooms({ silent: true })`
→ 다시 연결 검사와 전체 목록 조회
→ state 변경 및 재렌더링

탭이 숨겨지면 interval 자체는 살아 있지만 콜백이 요청을 건너뛴다. 탭이 다시 보일 때 `visibilitychange`로 즉시 한 번 갱신한다.

### 4) 정확히 어디가 문제인가?

`refreshRooms()`는 변경된 방만 받는 것이 아니라 매번 `GET /api/rooms` 전체 응답으로 `rooms`를 교체한다. Socket.IO가 이미 `roomCreated`, `roomUpdated`, `roomActivity`를 처리하므로 정합성 보정 목적의 전체 Polling 비용이 중복된다. 또한 `loadRooms()`는 매 주기 `attemptConnection()`을 먼저 기다린다.

다만 다음 완화책도 실제 코드에 존재한다.

- 숨김 탭에서는 요청하지 않는다.
- `isLoadingRef`로 겹친 목록 요청을 차단한다.
- interval과 `visibilitychange` listener를 cleanup한다.
- Socket.IO로 일부 변경을 증분 반영한다.

따라서 “모든 로그인 사용자가 무조건 30초마다 요청”은 정확하지 않다. 목록 화면에 있고 연결되었으며 탭이 보이는 사용자가 대상이다.

### 5) 데이터가 많아지면 어떻게 되는가?

방 수가 적고 동시 접속자가 적으면 응답 파싱과 테이블 갱신이 짧아 눈에 띄지 않는다. 방 수가 커지면 매 응답의 JSON 크기, 배열 교체, 테이블 reconciliation 비용이 커진다. 동시에 목록을 보는 사용자가 늘면 동일한 30초 주기의 요청 수가 선형으로 증가한다. interval 시작 시점이 비슷한 사용자가 많으면 요청이 특정 시점에 몰릴 가능성도 있다. 이는 추정이며 Network 및 서버 로그로 확인해야 한다.

### 6) 브라우저와 서버에는 어떤 영향을 주는가?

- 프론트엔드: 네트워크 사용량, JSON 파싱, React render가 주기적으로 발생한다.
- 백엔드: 목록 API RPS와 응답 전송량이 사용자 수에 비례해 늘 수 있다.
- Socket.IO: 증분 이벤트와 전체 Polling을 동시에 운영하는 비용이 발생한다.

### 7) 사용자에게는 어떻게 보이는가?

규모가 작으면 거의 보이지 않는다. 응답이 크거나 서버가 혼잡하면 목록 갱신 시점에 네트워크가 바빠지고 목록 컴포넌트가 다시 그려질 수 있다. 서버 부하가 누적되면 방 목록이 늦게 뜨거나 새로고침이 지연될 수 있다.

### 8) 어떻게 성능 테스트해야 하는가?

- Chrome DevTools Network에서 `/api/rooms`만 필터링해 최초 요청과 이후 요청 간격, 전송 크기, 응답 시간을 기록한다.
- 탭을 2분 이상 보이는 상태와 숨긴 상태로 각각 두어 요청 횟수를 비교한다.
- 여러 브라우저 세션을 동시에 열고 서버 access log 또는 APM에서 목록 API RPS를 본다.
- React DevTools Profiler로 30초 갱신 시 `ChatRoomsView`와 `RoomsTable` render 횟수·duration을 본다.
- k6로 백엔드 목록 API를 시험할 때 프론트의 실제 주기와 동시 사용자를 모델링하되, 프론트 측 Network 결과와 요청 헤더를 먼저 확인한다.

### 9) 개선 방법

1순위 개선: Socket.IO의 방 변경 이벤트를 정합성 있는 증분 동기화 수단으로 만들고, 전체 Polling은 재연결·포커스 복귀·수동 새로고침 등 보정 시점으로 축소한다. 서버 이벤트 유실과 순서 역전 대책이 필요하다.

2순위 개선: Polling이 필요하면 서버가 제공하는 변경 버전, ETag, `updatedSince` 등을 사용해 변경분만 받으며 interval에 jitter와 backoff를 둔다. 탭 간 중복 요청을 줄이기 위해 캐시 계층도 고려한다.

3순위 개선: 응답을 페이지네이션하고 동일 데이터일 때 state 교체를 피한다. 단, 활동도처럼 시간 경과로 바뀌는 표시의 갱신 요구사항을 먼저 정의해야 한다.

### 10) 개선 전 / 개선 후 예상 구조

개선 전

페이지 진입
→ 전체 방 조회
→ Socket.IO 증분 갱신
→ 30초
→ 전체 방 조회
→ 전체 state 교체

개선 후

페이지 진입
→ 최초 방 조회
→ Socket.IO 변경 이벤트 수신
→ 변경된 방만 갱신
→ 재연결 또는 포커스 복귀 시 버전 기반 보정 조회

### 11) 개선 효과를 어떻게 검증할까?

동일한 방 수, 사용자 수, 관찰 시간, 탭 visibility 조건으로 Before와 After를 측정한다. `/api/rooms` 요청 횟수·전송 bytes·응답 시간, 서버 RPS, `RoomsTable` render 횟수·duration을 비교하고 방 생성·수정·활동도가 누락 없이 반영되는지도 기능 지표로 함께 확인한다.

### 12) 핵심 정리

> 핵심: 보이는 방 목록 탭은 Socket.IO 증분 갱신과 별개로 30초마다 전체 목록을 조회한다. 우선 실제 RPS와 전송량을 측정한 뒤, 이벤트 기반 갱신과 버전 기반 보정 조회로 전체 Polling 빈도를 낮추는 것이 핵심이다.

---

## 2.2 방 입장 중복 요청

### 1) 이 문제가 무엇인가?

방 입장 중복 요청은 한 번의 사용자 동작으로 같은 목적의 API나 Socket.IO 이벤트를 여러 번 보내는 문제다. 이 프로젝트에서는 목록에서 REST로 멤버십 가입을 처리한 뒤 채팅 페이지에서 상세 REST 조회와 Socket.IO room 참가를 수행한다. 요청이 여러 개라는 사실만으로 중복은 아니다. REST 가입은 영속 멤버십, 상세 조회는 화면 데이터, Socket.IO join은 실시간 이벤트 구독이라는 서로 다른 책임일 수 있다.

### 2) 실제 코드 위치

- `features/chat/rooms/useRoomList.js:129-163`: `handleJoinRoom()`, `POST /api/rooms/:roomId/join` 후 route 이동
- `pages/chat/new.js:30-33, 63-70`: 새 방 생성 후 같은 REST join 실행
- `features/chat/room/useRoomHandling.js:193-233`: `fetchRoomData()`, `GET /api/rooms/:roomId`
- `features/chat/room/useRoomHandling.js:235-250`: `joinRoom()`, Socket.IO `joinRoomAndWait`
- `features/chat/room/useRoomHandling.js:280-325`: 조건부 `fetchPreviousMessagesAndWait`
- `features/chat/room/useRoomHandling.js:328-381`: 전체 `setupRoom()` 흐름과 `setupPromiseRef` 중복 실행 방지
- `lib/socket/socketClient.js:139-147`: `joinRoom`, `joinRoomSuccess`, `joinRoomError`

```javascript
await axiosInstance.post(`/api/rooms/${roomId}/join`, {});
router.push(`/chat/${roomId}`);

const roomData = await fetchRoomData(roomId);
const joinResult = await joinRoom(roomId);

if (Array.isArray(joinResult?.messages)) {
  processMessages(joinResult.messages, joinResult.hasMore, true);
} else {
  await loadInitialMessages(roomId);
}
```

### 3) 현재 코드 실행 흐름

기존 방 목록에서 입장 클릭
→ REST `POST /api/rooms/:id/join`
→ 성공 시 `/chat/:id` 이동
→ `setupRoom()`
→ Socket 연결
→ REST `GET /api/rooms/:id`
→ room 이벤트 listener 등록
→ Socket.IO `joinRoom`
→ join 응답에 `messages` 배열이 있으면 그대로 사용
→ 배열이 없을 때만 `fetchPreviousMessages` 실행

`setupPromiseRef`가 진행 중인 setup Promise를 재사용하므로 lifecycle effect와 connect handler가 겹쳐도 동일 setup을 동시에 반복하는 것은 방지된다.

### 4) 정확히 어디가 문제인가?

확인된 것은 한 번의 입장에 네트워크 왕복이 여러 단계로 이어진다는 점이다. 하지만 현재 프론트 코드만으로 REST join과 Socket.IO join이 서버에서 동일 작업을 중복 수행한다고 단정할 수 없다. 또한 상세 조회와 join 응답의 payload가 일부 겹칠 가능성은 있으나 응답 스키마의 완전한 중복 여부는 백엔드 확인이 필요하다.

초기 메시지에 대해서는 오히려 중복 방지 코드가 있다. join 결과에 `messages`가 있으면 별도 이전 메시지 요청을 하지 않는다. 재연결에서는 `rejoinRoom()`만 수행해 상세 REST 재조회도 피한다. 따라서 판정은 “부분 확인됨”이다.

### 5) 데이터가 많아지면 어떻게 되는가?

입장 사용자가 적을 때 여러 왕복은 개인의 latency로만 보인다. 입장 동시성이 커지면 REST 가입, 상세 조회, Socket.IO join이 각각 서버 작업을 만든다. 방 상세나 join payload가 큰 경우 네트워크와 직렬화 비용도 커진다. 다만 실제 중복 DB 작업 여부와 비용은 프론트 코드만으로 알 수 없다.

### 6) 브라우저와 서버에는 어떤 영향을 주는가?

- 프론트엔드: 방 화면이 준비될 때까지 여러 네트워크 왕복을 기다린다.
- 백엔드: 입장당 최소 REST 가입, 상세 조회, Socket.IO join을 처리한다.
- Socket.IO: 최초 입장과 재연결 입장을 구분해 계측해야 한다.

### 7) 사용자에게는 어떻게 보이는가?

네트워크 RTT가 크면 입장 클릭 후 로딩 화면이 오래 보일 수 있다. 한 단계가 실패하면 가입은 되었지만 화면 진입은 실패하는 식의 부분 성공도 가능하다. 이는 가능한 시나리오이며 실제 서버의 멱등성·트랜잭션 정책 확인이 필요하다.

### 8) 어떻게 성능 테스트해야 하는가?

- DevTools Network와 Socket.IO 계측을 동시에 켜고 입장 한 번당 REST URL과 socket event 수를 기록한다.
- `performance.mark()`로 클릭, REST join 완료, route 완료, 상세 완료, socket join 완료, 첫 메시지 표시 시점을 표시한다.
- 백엔드 로그에 correlation ID를 전달해 한 입장 동작의 DB 쿼리와 이벤트를 연결한다.
- 일반 입장, 이미 가입된 방 재입장, 새 방 생성, 소켓 재연결을 별도 시나리오로 측정한다.

### 9) 개선 방법

1순위 개선: REST join과 Socket.IO join의 서버 책임 및 응답 payload를 명확히 하고, 중복 데이터·DB 작업이 실제로 있는지 correlation trace로 확인한다. 의미가 다르면 무리하게 합치지 않는다.

2순위 개선: REST join 성공 응답이 완전한 방 상세를 제공한다면 route state 또는 캐시로 넘겨 즉시 표시하고, 상세 GET은 stale-while-revalidate로 바꿀 수 있다. 민감하거나 오래된 데이터를 신뢰하지 않도록 주의한다.

3순위 개선: 최초 socket join 응답에 초기 메시지와 필요한 room snapshot을 일관되게 포함하거나, 반대로 join을 구독 전용으로 가볍게 만든다. payload 크기와 권한 검증 위치를 함께 설계한다.

### 10) 개선 전 / 개선 후 예상 구조

개선 전

REST 멤버십 가입
→ route 이동
→ 방 상세 GET
→ Socket.IO join
→ join 결과에 메시지가 없으면 메시지 조회

개선 후 예시

REST 멤버십 가입 및 최소 room snapshot 수신
→ 즉시 route와 기본 화면 표시
→ Socket.IO 구독
→ 필요한 변경분만 보정

### 11) 개선 효과를 어떻게 검증할까?

같은 네트워크 throttling과 같은 방 데이터로 Before/After를 측정한다. 입장당 REST 요청 수, Socket.IO event 수, 전송 bytes, 클릭부터 방 헤더와 첫 메시지 표시까지의 duration, 서버 쿼리 수를 비교한다. 요청 감소와 함께 권한 검증 및 재연결 기능이 유지되는지도 확인한다.

### 12) 핵심 정리

> 핵심: 입장 한 번에 여러 요청이 발생하는 것은 확인되지만 각 요청의 책임이 달라 완전한 중복으로 단정할 수 없다. 서버 trace로 겹치는 작업과 payload를 찾고, 이미 받은 snapshot을 재사용하는 방향이 우선이다.

---

## 2.3 방 설정 순차 실행

### 1) 이 문제가 무엇인가?

서로 의존하지 않는 비동기 작업을 차례로 `await`하면 총 대기 시간이 각 작업의 합에 가까워진다. 병렬로 시작하면 보통 더 오래 걸린 작업의 시간에 가까워질 수 있다. 이 프로젝트의 방 초기화는 소켓 연결을 완전히 기다린 다음 방 상세 REST 조회를 시작한다.

### 2) 실제 코드 위치

- `features/chat/room/useRoomHandling.js:136-191`: `setupSocket()`
- `features/chat/room/useRoomHandling.js:193-233`: `fetchRoomData()`
- `features/chat/room/useRoomHandling.js:328-381`: `setupRoom()`의 순차 `await`
- `features/chat/room/useChatRoomLifecycle.js:181-217`: `initializeChat()`에서 `setupRoom()` 호출

```javascript
attachSocket(await setupSocket());
const roomData = await fetchRoomData(roomId);
setupEventListeners();
const joinResult = await joinRoom(roomId);
```

### 3) 현재 코드 실행 흐름

채팅 페이지 mount
→ 사용자 상태 확인
→ `setupRoom()`
→ Socket.IO 연결 완료 대기
→ `activeSocket` 연결
→ 방 상세 REST 완료 대기
→ 참여자 보정
→ 이벤트 listener 등록
→ Socket.IO join 완료 대기
→ join에 메시지가 없으면 초기 메시지 완료 대기
→ `setupSucceeded(roomData)`
→ 로딩 화면 종료

### 4) 정확히 어디가 문제인가?

`setupSocket()`과 `fetchRoomData(roomId)`는 모두 인증 정보와 room ID를 사용하지만, 프론트 코드상 방 상세 GET이 소켓 연결 결과를 필요로 하지는 않는다. 그런데 소켓 연결 후 상세 요청을 시작하므로 두 작업의 latency가 누적된다. 반면 listener 등록과 Socket.IO join은 소켓이 필요하고, 성공 상태 확정에는 room data가 필요하므로 모두 무조건 병렬화할 수 있는 것은 아니다.

또한 기존 소켓 정리 경로에서는 leave 후 1초, disconnect 후 2초를 순차 대기한다. 최초 진입에는 기존 소켓이 없을 가능성이 높지만 소켓 교체 상황에서는 의도적인 지연이 추가된다.

### 5) 데이터가 많아지면 어떻게 되는가?

데이터 양보다는 네트워크 RTT와 서버 응답 시간이 커질수록 직렬 대기의 영향이 커진다. 방 상세 payload가 크거나 소켓 handshake가 느리면 둘을 차례로 기다리는 시간이 눈에 띈다. 정확한 개선 시간은 측정 전에는 알 수 없다.

### 6) 브라우저와 서버에는 어떤 영향을 주는가?

- 프론트엔드: CPU 문제보다는 방 준비까지의 wall-clock latency가 늘어난다.
- 백엔드: 요청 수 자체는 같으나 병렬화 시 순간 동시 작업은 늘 수 있다.
- Socket.IO: handshake 및 join 지연이 전체 로딩 경로의 critical path가 된다.

### 7) 사용자에게는 어떻게 보이는가?

“채팅방 연결 중...” 화면이 길게 유지되고 첫 메시지 표시가 늦어진다. 느린 모바일 네트워크나 원거리 접속에서 더 분명해질 수 있다.

### 8) 어떻게 성능 테스트해야 하는가?

- DevTools Network waterfall에서 socket handshake와 `GET /api/rooms/:id`의 시작·종료가 겹치는지 확인한다.
- Performance API로 `setup-start`, `socket-ready`, `room-data-ready`, `join-ready`, `first-message-render`를 측정한다.
- Fast 3G 등 동일 throttling에서 여러 번 반복하고 중앙값과 상위 percentile을 비교한다.
- 실패·토큰 갱신·재연결 경로도 별도로 시험한다.

### 9) 개선 방법

1순위 개선: `setupSocket()`과 `fetchRoomData()`를 동시에 시작하고 `Promise.all` 또는 단계별 Promise로 합류시킨다. 한쪽 실패 시 다른 작업의 취소·정리와 오류 메시지를 설계해야 한다.

2순위 개선: socket 준비 즉시 listener를 먼저 등록해 join 직전 이벤트 유실 구간을 줄이고, room data 준비와 join 결과를 합류시킨다. listener가 불완전한 state를 읽지 않도록 주의한다.

3순위 개선: 기존 소켓 교체의 고정 1초·2초 대기를 ack 또는 실제 disconnect 상태 기반으로 바꿀 수 있는지 검토한다. 서버 정리 보장을 훼손해서는 안 된다.

### 10) 개선 전 / 개선 후 예상 구조

개선 전

Socket 연결 완료
→ 방 상세 GET 완료
→ listener 등록
→ Socket join 완료
→ 화면 표시

개선 후

Socket 연결 시작 + 방 상세 GET 시작
→ Socket 준비 즉시 listener 등록 및 join
→ room data와 join 결과 합류
→ 화면 표시

### 11) 개선 효과를 어떻게 검증할까?

동일한 네트워크와 서버 조건에서 클릭부터 socket-ready, room-data-ready, join-ready, first content까지의 시간을 Before/After로 비교한다. 평균만 보지 말고 실패율과 상위 percentile을 확인하며, 병렬화 후 서버 순간 부하와 취소되지 않은 요청도 살핀다.

### 12) 핵심 정리

> 핵심: 현재는 독립 가능한 소켓 연결과 방 상세 조회가 직렬 실행되어 입장 critical path가 길어진다. 두 작업을 병렬 시작하되 실패 정리와 권한 검증을 보존하는 것이 가장 중요한 개선 방향이다.

---

## 2.4 전체 메시지 계속 정렬/렌더링

### 1) 이 문제가 무엇인가?

메시지가 추가될 때마다 누적된 전체 배열을 정렬하면 `O(N log N)` 연산이 반복된다. 그 배열 전체를 `map`해 DOM으로 유지하면 메시지 수 `N`에 비례해 React 요소와 DOM node, 메모리가 늘어난다. Virtualization은 화면 주변 항목만 DOM에 두어 이 비용을 제한하는 기법이다.

### 2) 실제 코드 위치

- `features/chat/messages/useMessageList.js:1-38`: 신규·과거 메시지 병합 후 전체 `.sort()`, 다시 `Map` 순회
- `features/chat/room/roomEventHandlers.js:67-73`: 실시간 메시지는 append
- `components/ChatMessages.js:66-73`: `messages` 변경마다 전체 복사·정렬
- `components/ChatMessages.js:121-128`: 전체 `allMessages.map()` 렌더
- `features/chat/room/useMessageHandling.js:47-54`: 더 불러올 때 가장 오래된 메시지를 찾기 위해 다시 전체 정렬
- `components/ChatMessages.js:93-96`: `contentVisibility: auto` 완화책

```javascript
const allMessages = useMemo(() => {
  return [...messages].sort((a, b) => {
    return new Date(a.timestamp) - new Date(b.timestamp);
  });
}, [messages]);

allMessages.map((msg, idx) => renderMessage(msg, idx));
```

### 3) 현재 코드 실행 흐름

새 메시지 수신
→ 중복 ID 확인
→ 기존 배열 끝에 append
→ `messages` 참조 변경
→ `ChatMessages`의 `useMemo` 전체 정렬
→ 전체 메시지 `map`
→ React reconciliation
→ 모든 메시지 wrapper를 DOM에 유지

과거 메시지 로드
→ 기존+수신 메시지 전체 정렬
→ 전체 `Map` 중복 정리
→ state 갱신
→ 표시 단계에서 다시 전체 정렬
→ 전체 렌더 목록 생성

### 4) 정확히 어디가 문제인가?

과거 메시지 병합에서는 전체 정렬과 전체 Map 순회가 있고, state가 바뀌면 `ChatMessages`가 같은 데이터를 다시 정렬한다. 새 실시간 메시지는 append만 하지만 표시 단계의 전체 정렬은 피하지 못한다. `React.memo`, `useMemo`, `useCallback`이 사용되었어도 `messages` 배열은 새 참조이므로 정렬 memo는 다시 계산된다. 자식 props인 `room`, callbacks 등의 참조가 바뀌면 개별 메시지 컴포넌트도 재렌더될 수 있다.

`contentVisibility: auto`는 화면 밖 painting/layout 비용을 줄일 수 있으나 React 요소 생성과 DOM node 누적 자체를 제거하는 virtualization은 아니다.

### 5) 데이터가 많아지면 어떻게 되는가?

예시로 메시지 30개에서는 정렬과 DOM 유지 비용이 작다. 10,000개가 누적된 상황에서는 새 메시지 한 개로 10,001개 전체 정렬과 map이 다시 실행되고, 모든 wrapper가 DOM에 남는다. 이는 규모 설명용 예시이며 프로젝트의 실제 최대 메시지 수나 측정값이 아니다. 무한 스크롤로 과거 메시지를 계속 가져오므로 세션이 길수록 `N`이 증가한다.

### 6) 브라우저와 서버에는 어떤 영향을 주는가?

- JavaScript CPU와 scripting time 증가
- React render/reconciliation 증가
- DOM node와 JS heap 증가
- 긴 Main Thread 작업, 스크롤 끊김, FPS 저하 가능
- 이 항목은 서버 요청 증가보다 브라우저 비용이 중심이다.

### 7) 사용자에게는 어떻게 보이는가?

과거 메시지를 많이 불러온 뒤 새 메시지가 올 때 끊기거나, 스크롤과 반응 버튼이 늦게 반응할 수 있다. 오래 열린 탭의 메모리가 증가하고 저사양 기기에서 체감이 더 커질 수 있다.

### 8) 어떻게 성능 테스트해야 하는가?

- 30개, 1,000개, 10,000개 등 통제된 fixture로 메시지 수를 단계화한다. 큰 수는 테스트 데이터일 뿐 운영 수치가 아니다.
- React DevTools Profiler에서 새 메시지 1개당 commit 수, `ChatMessages`와 자식 render duration을 기록한다.
- Performance 패널에서 scripting, rendering, long task, FPS를 본다.
- Memory 패널에서 DOM node 수와 heap snapshot을 메시지 로드 전후로 비교한다.
- `performance.mark()`로 이벤트 수신부터 paint까지의 시간을 기록한다.

### 9) 개선 방법

1순위 개선: 메시지 virtualization을 적용해 화면과 overscan 범위만 DOM에 유지한다. 가변 높이, 위쪽 prepend 후 스크롤 위치 보존, IntersectionObserver 읽음 처리와의 통합이 핵심 주의점이다.

2순위 개선: 정렬된 상태를 state 불변식으로 만들고 실시간 메시지는 끝에 추가하며, 과거 페이지는 정렬된 병합으로 합친다. 표시 컴포넌트의 두 번째 전체 sort를 제거한다.

3순위 개선: 메시지 ID `Map/Set`과 가장 오래된 timestamp를 별도 유지하고, 메시지 행의 props와 callback 참조를 안정화한다. 순서가 뒤늦게 도착하는 이벤트를 올바른 위치에 삽입해야 한다.

### 10) 개선 전 / 개선 후 예상 구조

개선 전

새 메시지
→ 전체 배열 sort
→ 전체 메시지 map
→ 누적 DOM 유지

개선 후

새 메시지
→ 정렬 불변식을 지키며 append 또는 이진 삽입
→ 화면 주변 행만 render
→ virtualization 범위 밖 DOM 제거

### 11) 개선 효과를 어떻게 검증할까?

같은 메시지 fixture, viewport, 스크롤 위치, 새 메시지 발생률로 Before/After를 비교한다. 전체 sort 호출 수, commit 수와 duration, scripting/layout time, long task, FPS, DOM node 수, JS heap, 이벤트부터 paint까지의 지연을 기록한다. virtualization 후 스크롤 점프와 읽음 누락도 회귀 검사한다.

### 12) 핵심 정리

> 핵심: 누적 메시지는 병합과 표시 과정에서 전체 정렬되고 모두 DOM에 남는다. 정렬된 데이터 구조를 한 곳에서 유지하고 virtualization으로 실제 DOM 수를 제한하는 것이 가장 큰 개선 효과를 낼 가능성이 높다.

---

## 2.5 읽음 이벤트 배열 탐색

### 1) 이 문제가 무엇인가?

배열 탐색은 한 번이면 대개 빠르지만, 한 배열의 각 원소마다 다른 배열을 다시 탐색하면 `O(N×M)`이 된다. 이 프로젝트는 읽음 receipt를 메시지에 반영할 때 메시지 전체를 돌며 `messageIds.includes()`를 호출한다. 각 메시지의 읽지 않은 인원 계산도 참여자마다 readers를 `.some()`으로 찾는다.

### 2) 실제 코드 위치

- `features/chat/room/roomEventHandlers.js:35-55`: `applyReadReceipts()`
- `features/chat/room/roomEventHandlers.js:91-94`: Socket.IO `onMessagesRead`에서 전체 state 갱신
- `components/ReadStatus.js:20-30`: `participants.filter` 안의 `readers.some`
- `components/ReadStatus.js:61-95`: 메시지별 `IntersectionObserver`, 이미 읽음 검사, `markMessagesAsRead([messageId])`
- `components/UserMessage.js:85`, `components/FileMessage.js:366`: 각 메시지에서 `ReadStatus` 렌더

```javascript
messages.map((msg) => {
  if (!messageIds.includes(msg._id)) return msg;

  const alreadyRead = msg.readers?.some(
    (reader) => reader.userId === userId || reader._id === userId
  );
  // ...
});
```

### 3) 현재 코드 실행 흐름

메시지가 50% 이상 viewport에 노출
→ 해당 메시지의 `IntersectionObserver` callback
→ `markMessagesAsRead([messageId])` Socket.IO 이벤트 전송
→ 서버가 `messagesRead` payload 송신
→ `applyReadReceipts(prev, payload)`
→ 전체 메시지 `map`
→ 각 메시지마다 `messageIds.includes`
→ 대상 메시지는 `readers.some`
→ 새 messages 배열로 전체 채팅 렌더 경로 재실행
→ 각 표시 메시지의 `ReadStatus`에서 참여자별 `readers.some`

### 4) 정확히 어디가 문제인가?

메시지 수를 `N`, 한 receipt의 ID 수를 `K`라 하면 `messages.map`과 `messageIds.includes` 조합은 최악 `O(N×K)`이다. 대상 메시지별 중복 reader 확인 비용도 추가된다. 각 메시지 UI에서 참여자 수 `P`, readers 수 `R`에 대해 `participants.filter(...readers.some...)`가 `O(P×R)`이며, receipt가 state 배열을 바꾸면 여러 `ReadStatus`가 다시 계산될 수 있다.

또한 메시지마다 별도의 `IntersectionObserver` 인스턴스를 만든다. cleanup은 존재하지만 메시지 DOM 수가 많으면 observer 수도 많아진다. 이벤트가 메시지별 단건 배열로 자주 발행될 가능성이 있으며, 실제 batching 여부는 서버와 실행 계측이 필요하다.

### 5) 데이터가 많아지면 어떻게 되는가?

메시지와 참여자가 적으면 짧은 배열 탐색이라 보이지 않는다. 누적 메시지 `N`, receipt batch `K`, 참여자 `P`, readers `R`이 커지면 한 이벤트가 만드는 비교 횟수가 곱으로 증가한다. 화면에 많은 메시지가 동시에 나타나면 단건 읽음 event가 연속 발생할 수도 있다. 이는 코드 기반 위험이며 실제 event 빈도는 측정해야 한다.

### 6) 브라우저와 서버에는 어떤 영향을 주는가?

- JavaScript CPU와 React render 증가
- 많은 observer와 메시지 state로 메모리 증가
- 연속 receipt 처리 시 Main Thread 작업과 스크롤 끊김 가능
- 서버에는 Socket.IO 읽음 event 수와 broadcast 수 증가 가능

### 7) 사용자에게는 어떻게 보이는가?

참여자와 메시지가 많은 방에서 스크롤할 때 읽음 표시가 늦게 변하거나 스크롤이 끊길 수 있다. 다른 사용자의 읽음이 몰리면 반응 버튼이나 입력 반응이 늦어질 수 있다.

### 8) 어떻게 성능 테스트해야 하는가?

- 메시지 수, 참여자 수, receipt batch 크기를 독립적으로 바꾼 fixture를 만든다.
- Socket.IO 송수신 event 이름, 초당 수, payload의 message ID 수를 기록한다.
- React Profiler로 한 `messagesRead`당 render/commit 수와 duration을 본다.
- Performance 패널에서 receipt burst 구간의 scripting과 long task를 본다.
- 브라우저 콘솔 계측은 개발 전용으로 넣고 운영 코드에는 남기지 않는다.

### 9) 개선 방법

1순위 개선: `messageIds`를 한 번 `Set`으로 바꾸어 membership 검사를 `O(1)` 평균으로 만들고, ID 기반 메시지 index 또는 normalized state로 대상 메시지만 갱신한다.

2순위 개선: 참여자 ID와 reader ID를 `Set`으로 정규화해 unread 계산을 `O(P+R)`로 줄이며, room 참여자 변경과 receipt 변경 시에만 재계산한다.

3순위 개선: 보이는 메시지의 읽음 전송을 짧은 debounce/batch로 묶고 공유 IntersectionObserver 또는 virtualization과 결합한다. 읽음 의미와 전송 지연 허용 범위를 제품 요구사항으로 정해야 한다.

### 10) 개선 전 / 개선 후 예상 구조

개선 전

읽음 event
→ 전체 메시지 map
→ 매 메시지마다 ID 배열 탐색
→ 각 메시지에서 참여자×reader 탐색

개선 후

읽음 ID batch를 Set으로 변환
→ ID index로 대상 메시지만 갱신
→ reader Set으로 unread 계산
→ 필요한 메시지 행만 render

### 11) 개선 효과를 어떻게 검증할까?

같은 `N`, `K`, `P`, `R`과 동일한 receipt 발생 순서로 Before/After를 재생한다. event 수와 payload 크기, handler scripting time, render 횟수·duration, long task, heap, 스크롤 FPS를 비교한다. 중복 receipt와 이미 읽은 메시지가 정확히 멱등 처리되는지도 확인한다.

### 12) 핵심 정리

> 핵심: 읽음 반영은 전체 메시지×ID 배열 탐색이고, 표시 계산은 참여자×reader 탐색이다. ID Set과 normalized index로 대상만 갱신하고, 읽음 전송을 안전하게 batch하는 것이 핵심이다.

---

## 2.6 입력마다 DOM/레이아웃 계산

### 1) 이 문제가 무엇인가?

브라우저는 DOM과 CSS를 바탕으로 요소의 크기와 위치를 계산한다. JavaScript가 style을 바꾼 직후 `offsetWidth`, `getBoundingClientRect`, `scrollHeight` 같은 값을 읽으면 최신 값을 주기 위해 동기 Layout이 발생할 수 있다. 이를 흔히 forced reflow라고 부른다.

이 프로젝트의 일반 텍스트 입력에서는 직접 `scrollHeight`를 읽는 코드를 찾지 못했다. 다만 `Textarea autoResize={true}`가 사용되며 라이브러리 내부 구현은 현재 프로젝트 소스만으로 검증할 수 없다. 명확히 확인된 문제는 `@` 뒤에 공백 없이 입력하는 멘션 모드에서 매 입력마다 임시 DOM을 만들고 여러 layout/style 값을 읽는 경로다.

### 2) 실제 코드 위치

- `components/ChatInput.js:187-249`: `calculateMentionPosition()`
- `components/ChatInput.js:251-276`: `handleInputChange()`에서 활성 멘션마다 위치 계산
- `components/ChatInput.js:408-420`: 제어 `Textarea`, `autoResize={true}`
- `features/chat/composer/useMessageComposer.js`: 입력·멘션 관련 state 관리

```javascript
document.body.appendChild(measureDiv);
const textWidth = measureDiv.offsetWidth;
document.body.removeChild(measureDiv);

const textareaRect = textarea.getBoundingClientRect();
const computedStyle = window.getComputedStyle(textarea);
const scrollTop = textarea.scrollTop;
```

### 3) 현재 코드 실행 흐름

일반 문자 입력
→ `onChange`
→ `setMessage(value)`
→ `ChatInput` 재렌더
→ `Textarea` 갱신
→ `autoResize` 내부 동작 가능성은 현재 코드만으로 미확인

활성 멘션 입력
→ `@` 이후 문자 입력
→ 여러 mention state 갱신
→ hidden div 생성 및 style 설정
→ body에 삽입
→ `offsetWidth` 측정
→ textarea의 rect와 computed style 반복 조회
→ dropdown 위치 state 갱신
→ `ChatInput`과 `MentionDropdown` 렌더

### 4) 정확히 어디가 문제인가?

`calculateMentionPosition()`은 호출마다 DOM node를 생성·삽입·삭제하고, 삽입 직후 `offsetWidth`를 읽는다. 이어 `getBoundingClientRect`, 여러 번의 `getComputedStyle`, `scrollTop`을 읽는다. style/DOM 변경 뒤 geometry를 읽는 패턴은 동기 layout 가능성이 있다. 특히 `window.getComputedStyle(textarea)`를 속성별로 반복 호출한다.

반면 “모든 키 입력마다 직접 textarea의 scrollHeight를 측정하고 height를 쓴다”는 코드는 현재 저장소에서 확인되지 않았다. 따라서 전체 항목은 부분 확인이다.

### 5) 데이터가 많아지면 어떻게 되는가?

이 문제는 메시지 수보다는 입력 빈도, textarea 주변 DOM 복잡도, 참여자 필터링 크기와 관련된다. 짧은 일반 입력에서는 해당 측정 경로가 실행되지 않는다. 긴 멘션을 빠르게 입력하거나 페이지 layout이 복잡할수록 동기 측정 비용이 커질 수 있다. 실제 forced reflow 여부와 시간은 Performance trace가 필요하다.

### 6) 브라우저와 서버에는 어떤 영향을 주는가?

- React render와 JavaScript 실행 증가
- DOM 삽입/삭제와 Layout/Reflow 증가 가능
- Main Thread 점유, 입력 지연, FPS 저하 가능
- 서버에는 직접 영향이 없다.

### 7) 사용자에게는 어떻게 보이는가?

특히 `@이름`을 빠르게 입력할 때 글자가 늦게 나타나거나 멘션 목록이 떨리고, 모바일에서 키 입력이 버벅일 수 있다. 일반 입력까지 느린지는 별도로 측정해야 한다.

### 8) 어떻게 성능 테스트해야 하는가?

- Performance 패널에서 일반 문장과 `@` 멘션 문장을 동일 타수로 입력해 Event, Scripting, Layout, Recalculate Style을 비교한다.
- DevTools Rendering의 FPS meter와 Performance의 long task를 확인한다.
- React Profiler로 키 입력당 `ChatInput`, `MentionDropdown` render 횟수와 duration을 기록한다.
- textarea 높이가 변하는 여러 줄 입력에서 autoResize의 layout 비용을 별도 확인한다.
- CPU throttling으로 저사양 환경을 재현하되 결과에 throttling 배수를 명시한다.

### 9) 개선 방법

1순위 개선: textarea computed style을 한 번 읽어 캐시하고, 재사용 가능한 off-screen 측정 요소를 유지하며, DOM write와 read를 한 프레임 안에서 묶어 layout thrashing을 줄인다. 폰트·resize 변경 시 캐시 무효화가 필요하다.

2순위 개선: 멘션 위치 갱신을 `requestAnimationFrame` 단위로 throttle하고 동일 프레임의 중복 계산을 합친다. cursor가 이동하거나 scroll/resize될 때의 정확성을 유지해야 한다.

3순위 개선: 텍스트 caret 위치 전용 라이브러리나 단순 anchor UI를 검토하고, `Textarea`의 autoResize 실제 구현을 프로파일링한 뒤 필요할 때만 debounce 또는 최대 높이를 적용한다. 입력 state 자체를 과도하게 debounce하면 화면과 값이 어긋날 수 있다.

### 10) 개선 전 / 개선 후 예상 구조

개선 전

멘션 문자 입력
→ 임시 DOM 생성·삽입
→ offsetWidth 및 rect/style 동기 읽기
→ DOM 삭제
→ 위치 state 갱신

개선 후

멘션 문자 입력
→ 최신 입력값 저장
→ animation frame당 한 번
→ 재사용 측정 요소와 캐시 style로 위치 계산
→ 필요한 경우만 위치 state 갱신

### 11) 개선 효과를 어떻게 검증할까?

동일 문자열, 입력 속도, 참여자 수, viewport, CPU throttling으로 Before/After trace를 수집한다. 키 입력당 render 수, scripting time, style recalculation, layout 횟수·시간, long task, FPS, INP에 가까운 event duration을 비교한다. 멘션 위치 정확성과 IME 한글 입력도 함께 회귀 검사한다.

### 12) 핵심 정리

> 핵심: 모든 일반 입력의 직접 `scrollHeight` 계산은 확인되지 않았지만, 활성 멘션 입력은 매 키마다 DOM 삽입과 geometry 측정을 수행한다. 측정 요소·스타일 캐시와 frame 단위 throttle로 동기 layout 가능성을 줄여야 한다.

---

## 2.7 파일 Preview 메모리 관리

### 1) 이 문제가 무엇인가?

`URL.createObjectURL(file)`은 Blob/File을 가리키는 브라우저 내부 URL을 만든다. 더 이상 필요 없을 때 `URL.revokeObjectURL(url)`을 호출하지 않으면 페이지가 살아 있는 동안 Blob 메모리가 유지될 수 있다. 파일 크기와 선택 개수 제한도 한 번에 유지되는 메모리의 상한을 정하는 데 중요하다.

### 2) 실제 코드 위치

- `components/ChatInput.js:53-66`: preview Blob URL 생성과 files 배열 추가
- `components/ChatInput.js:80-85`: 개별 제거 시 revoke
- `components/ChatInput.js:115-125`: 제출 후 `files` 초기화
- `components/ChatInput.js:128-184`: effect cleanup에서 현재 파일 URL revoke
- `components/FilePreview.js:26-41, 349-363`: 내부 URL map의 unmount·제거 cleanup
- `features/chat/room/useFileHandling.js:74-102, 119-130, 159-166`: 별도 preview 경로의 생성·제거·effect cleanup
- `services/fileService.js:8-25, 29-75`: 전체 50MB, 이미지 10MB, PDF 20MB 및 형식 제한
- `components/ProfileImageUpload.js:41-50, 96-99, 127-130, 146-153`: 프로필 URL 생성 및 cleanup

```javascript
const filePreview = {
  file,
  url: URL.createObjectURL(file),
  name: file.name,
  type: file.type,
  size: file.size,
};

setFiles((prev) => [...prev, filePreview]);
```

### 3) 현재 코드 실행 흐름

채팅 파일 선택
→ `fileService.validateFile(file)`
→ Blob URL 생성
→ `files` 배열에 추가
→ `FilePreview`가 `file.url` 표시
→ 제거하면 즉시 revoke
→ 제출 후 `setFiles([])`
→ files 변경으로 이전 effect cleanup 실행
→ 이전 배열의 URL revoke
→ 언마운트 시에도 남은 URL revoke

파일 입력·drop·paste 한 번에는 첫 파일만 처리하지만, 사용자는 제출 전 선택을 반복해 배열에 계속 추가할 수 있다. `FilePreview`의 기본 `maxFiles=10`은 경고·내부 drop 제한에 쓰일 뿐 `ChatInput`의 반복 선택 추가를 막지 않는다.

### 4) 정확히 어디가 문제인가?

주요 채팅 경로의 URL cleanup은 확인되므로 “Blob URL을 해제하지 않는다”는 주장은 현재 코드에서는 사실이 아니다. 파일별 크기와 형식 제한도 있다. 다만 다음은 개선 여지가 있다.

- `ChatInput`의 배열 추가 자체에는 최대 개수 검사가 없다.
- URL 생성·revoke 책임이 `ChatInput`, `FilePreview`, `useFileHandling`, 프로필 컴포넌트에 분산돼 추적이 어렵다.
- `useFileHandling`은 `ChatRoomView`에서 사실상 `fileInputRef`만 소비되고 preview state는 실제 입력 UI와 분리돼 있어 중복·미사용 구조다.
- 같은 이름으로 제거할 때 `filter(file.name !== ...)`가 동명 파일을 모두 제거한다. 이는 주로 정확성 문제다.

cleanup effect가 `[files]`에 의존하므로 files가 바뀔 때 이전 배열 URL을 revoke한다. 현재는 기존 URL을 계속 표시해야 하는 배열 추가 직후에도 이전 URL이 revoke될 수 있어, 여러 파일 preview를 유지하려는 의도와 충돌할 가능성이 있다. 실제 이미지 표시 동작은 브라우저에서 검증해야 한다.

### 5) 데이터가 많아지면 어떻게 되는가?

파일 한 개는 제한 덕분에 상한이 있지만 반복 선택으로 여러 File과 Blob URL을 배열에 보관하면 메모리가 합산된다. 이미지 디코딩 시 원본 파일 크기보다 큰 메모리를 사용할 수도 있으나 정확한 배수는 형식과 브라우저에 따라 달라 수치를 단정할 수 없다. cleanup이 정상 동작하면 제거·제출·언마운트 후 회수 가능해야 한다.

### 6) 브라우저와 서버에는 어떤 영향을 주는가?

- 프론트엔드: Blob, 이미지 디코딩, preview DOM으로 heap 및 브라우저 메모리 증가 가능
- 오래 선택 상태를 유지하면 탭 메모리 증가 가능
- 업로드 시 네트워크 사용량이 증가하지만 크기 제한은 존재한다.
- 서버 영향은 업로드 요청의 크기와 동시성에 좌우된다.

### 7) 사용자에게는 어떻게 보이는가?

반복 선택 후 preview가 깨지거나 브라우저 메모리가 커질 수 있고, 큰 이미지 여러 개에서 탭이 느려질 수 있다. 현재 코드에서 지속 누수가 확정된 것은 아니므로 Memory 도구로 확인해야 한다.

### 8) 어떻게 성능 테스트해야 하는가?

- 작은 이미지, 허용 최대 크기 근처 이미지, PDF를 반복 선택·제거·제출한다.
- Chrome Memory의 heap snapshot과 Allocation instrumentation으로 File/Blob 관련 객체와 detached DOM을 확인한다.
- Chrome Task Manager의 탭 메모리와 `performance.memory`가 지원되는 환경의 JS heap 추이를 보조 지표로 기록한다.
- `createObjectURL`/`revokeObjectURL`을 테스트 환경에서 wrapper로 계측해 생성-해제 잔여 개수를 센다.
- 10개를 넘는 반복 선택, 동명 파일, 제출 실패, route 이동을 포함한다.

### 9) 개선 방법

1순위 개선: preview URL의 소유자를 하나의 hook으로 통합하고, 파일 고유 ID별 생성·교체·제거·제출·언마운트 cleanup 규칙을 명시한다. revoke된 URL을 렌더 중 재사용하지 않게 해야 한다.

2순위 개선: `ChatInput`의 추가 함수에서 최대 파일 개수와 총합 크기를 강제한다. 현재 전송 로직은 첫 파일만 사용하므로 제품 요구가 단일 파일이면 state도 단일 파일로 제한하는 편이 명확하다.

3순위 개선: 썸네일 크기를 제한하고 필요하면 이미지 리사이즈/디코딩 전략을 적용한다. 원본 업로드 품질과 EXIF, 브라우저 호환성을 고려한다.

### 10) 개선 전 / 개선 후 예상 구조

개선 전

여러 컴포넌트에서 URL 생성
→ files 배열 반복 추가 가능
→ effect·제거 handler·unmount에서 분산 revoke

개선 후

단일 preview hook
→ 개수·총합 크기 검증
→ 고유 ID별 URL 소유
→ 교체·제거·제출·언마운트에서 정확히 한 번 revoke

### 11) 개선 효과를 어떻게 검증할까?

동일 파일 세트와 동일 반복 횟수로 Before/After를 수행한다. create/revoke 횟수와 잔여 URL 수, 선택 전·후·정리 후 heap, 탭 메모리, DOM node 수, preview 오류 여부를 비교한다. GC 시점의 변동을 줄이기 위해 여러 회 반복하고 강제 GC가 가능한 DevTools 환경에서는 동일 절차를 사용한다.

### 12) 핵심 정리

> 핵심: 현재 주요 preview 경로에는 revoke와 크기 제한이 있어 명백한 무해제 누수는 확인되지 않는다. 다만 반복 선택 개수와 URL 소유권이 불명확하므로 단일 관리 hook과 명시적 상한으로 메모리 수명을 예측 가능하게 만들어야 한다.

---

## 3. 추가 개념 설명

### Polling

클라이언트가 일정 간격으로 서버에 “변경됐는가?”를 묻는 방식이다. 구현은 단순하지만 변경이 없어도 요청하며, 사용자 수에 비례해 서버 요청이 늘 수 있다.

### WebSocket / Socket.IO

WebSocket은 브라우저와 서버가 연결을 유지하며 양방향으로 메시지를 주고받는 프로토콜이다. Socket.IO는 그 위 또는 fallback transport 위에서 재연결, event 이름, room 같은 기능을 제공하는 라이브러리다. 연결이 있다고 해서 REST 멤버십과 Socket.IO room 참가가 같은 개념인 것은 아니다.

### React Re-render

state나 props 등이 바뀌어 컴포넌트 함수를 다시 실행하고 새 React element tree를 계산하는 과정이다. 재렌더가 곧 실제 DOM 전체 교체를 뜻하지는 않지만, 계산과 비교 비용은 발생한다.

### useEffect

렌더 결과가 반영된 뒤 네트워크 호출, timer, event listener 같은 외부 작업을 연결하는 Hook이다. dependency가 바뀌면 이전 cleanup 후 다시 실행될 수 있으므로 interval, listener, Blob URL의 수명 관리가 중요하다.

### Memoization

입력 값이 같을 때 이전 계산 결과나 함수·컴포넌트 결과를 재사용하는 기법이다. 캐시 비교와 메모리 비용도 있으므로 비싼 계산 또는 안정적인 props 경계에 사용한다.

### React.memo

부모가 재렌더돼도 props가 얕은 비교상 같으면 자식 함수 실행을 건너뛴다. 새 배열·객체·함수를 매번 전달하면 효과가 줄어든다.

### useMemo

dependency가 같을 때 계산 결과를 재사용한다. `messages` 배열이 새 참조가 되면 전체 정렬을 다시 하므로, `useMemo`만으로 데이터 증가 문제를 해결할 수 없다.

### useCallback

dependency가 같을 때 함수 참조를 재사용한다. memoized 자식에게 callback을 전달할 때 유용하지만 함수 내부 연산을 자동으로 빠르게 만들지는 않는다.

### Virtualization

긴 목록 전체가 아니라 viewport 주변 항목만 DOM에 렌더링하는 방식이다. DOM 수를 제한하지만 가변 높이, 스크롤 복원, 접근성, 읽음 observer 처리를 세심하게 통합해야 한다.

### DOM

브라우저가 HTML 요소를 객체 트리로 표현한 것이다. node가 많아지면 style 계산, layout, 메모리 비용이 커질 수 있다.

### Main Thread

대부분의 JavaScript 실행, style 계산, layout, paint 준비와 사용자 event 처리가 진행되는 주 스레드다. 긴 작업이 점유하면 입력과 화면 갱신이 기다린다.

### Reflow / Layout

요소의 크기와 위치를 계산하는 과정이다. DOM/style을 쓴 뒤 geometry를 즉시 읽는 패턴이 반복되면 동기 layout이 연쇄적으로 발생할 수 있다.

### Repaint

layout이 바뀌지 않아도 색상, 그림자 등 픽셀을 다시 그리는 과정이다. Layout과 별개지만 둘 다 프레임 예산을 사용할 수 있다.

### Long Task

Main Thread에서 일반적으로 50ms를 넘게 연속 실행되는 작업을 말한다. 그동안 입력과 paint가 지연될 수 있으며 DevTools Performance에서 확인할 수 있다.

### O(N)

데이터 `N`개를 한 번씩 보는 선형 복잡도다. 데이터가 두 배면 비교 횟수도 대략 두 배가 되는 형태다.

### O(N×M)

`N`개 각각에 대해 `M`개를 다시 탐색하는 형태다. 메시지마다 receipt ID 배열을 `includes`로 찾거나, 참여자마다 readers를 찾는 코드가 예다.

### Blob

브라우저 메모리에서 파일과 같은 바이너리 데이터를 표현하는 객체다. 이미지, PDF, 업로드 파일 preview에 사용된다.

### Blob URL

Blob을 브라우저 요소의 `src` 등에서 참조할 수 있게 만든 `blob:` 형식 URL이다. 일반 서버 URL이 아니며 해당 document의 수명과 명시적 revoke에 영향을 받는다.

### URL.createObjectURL

Blob/File을 가리키는 Blob URL을 생성한다. 호출할 때마다 새 URL이 생기므로 소유자와 수명 정책이 필요하다.

### URL.revokeObjectURL

더 이상 필요 없는 Blob URL을 해제한다. 이미지가 아직 그 URL을 사용 중일 때 너무 일찍 호출하면 preview가 깨질 수 있고, 너무 늦거나 누락되면 메모리 유지가 길어진다.

### Memory Leak

더 이상 필요 없는 객체가 reference, listener, timer, Blob URL 등 때문에 계속 도달 가능해 회수되지 않는 현상이다. 메모리가 한 번 증가했다는 사실만으로 누수는 아니며 반복 작업 후 기준선으로 돌아오는지 확인해야 한다.

### Debounce

연속 호출이 멈춘 뒤 일정 시간이 지나 한 번 실행하는 방식이다. 검색 요청 등에 적합하지만 입력 UI 자체를 debounce하면 화면 반응이 늦을 수 있다.

### Throttle

연속 호출 중 일정 시간 또는 프레임당 최대 한 번만 실행하는 방식이다. scroll, resize, caret 위치 계산처럼 지속적인 최신화가 필요하지만 매 event 실행은 비싼 작업에 적합하다.

---

## 4. 성능 개선 우선순위

| 순위 | 문제 | 이유 | 검증 방법 | 개선 난이도 |
| --- | --- | --- | --- | --- |
| 1 | 전체 메시지 계속 정렬/렌더링 | 코드에서 명확하고 데이터 증가에 따라 브라우저 CPU·DOM·메모리가 함께 커지며 재현이 쉬움 | React Profiler, Performance, Memory에서 메시지 수 단계별 비교 | 높음 |
| 2 | 읽음 이벤트 배열 탐색 | 중첩 탐색과 메시지별 observer가 명확하며 대형 방에서 메시지 렌더 병목을 증폭함 | receipt burst 재생, Socket.IO 계측, Profiler | 중간 |
| 3 | 방 목록 30초 Polling | 실제 존재하고 동시 사용자 증가 시 서버 RPS에 직접 연결되며 요청 횟수 측정이 쉬움 | Network와 서버 access log에서 30초 주기 및 RPS 확인 | 중간 |
| 4 | 방 설정 순차 실행 | 실제 critical path지만 요청 수보다 사용자 입장 latency가 중심이며 병렬화 효과 측정이 명확함 | Network waterfall과 Performance marks | 중간 |
| 5 | 입력마다 DOM/레이아웃 계산 | 멘션 입력에서 확인되지만 일반 입력 전체 문제는 아니며 국소 최적화 가능 | 일반 입력과 멘션 입력의 Performance trace 비교 | 낮음~중간 |
| 6 | 방 입장 중복 요청 | 여러 요청은 확인되지만 의미 중복 여부는 백엔드 trace 없이는 확정 불가 | REST+Socket correlation trace와 payload 비교 | 중간~높음 |
| 7 | 파일 Preview 메모리 관리 | 주요 cleanup과 크기 제한이 이미 있어 위험이 완화됐고, 실제 누수 여부를 먼저 입증해야 함 | create/revoke 계측과 heap snapshot | 낮음~중간 |

이 순위는 심각도만이 아니라 현재 코드에서의 확실성, 사용자·서버 영향, 재현성, 예상 개선 폭과 구현 위험을 함께 고려한 것이다.

---

## 5. 성능 테스트 TOP 3

## 5.1 전체 메시지 정렬·렌더링 테스트

### 테스트 대상

`deriveUniqueSortedMessages`, `ChatMessages`, 메시지 행 컴포넌트와 누적 DOM이다.

### 테스트 준비

동일한 timestamp 분포와 콘텐츠 형태를 가진 30개, 1,000개, 10,000개 테스트 fixture를 준비한다. 브라우저, viewport, CPU throttling, React 개발/프로덕션 빌드 조건을 고정하고 각각 별도로 기록한다.

### Before 측정 방법

각 규모에서 초기 표시, 과거 30개 prepend, 실시간 메시지 1개 append를 수행한다. React Profiler와 Performance trace를 동시에 수집하고 전후 heap snapshot과 DOM node 수를 기록한다.

### 확인할 지표

- 전체 sort 호출 및 소요 시간
- React commit과 메시지 행 render 횟수·duration
- scripting, rendering, layout, long task
- DOM node 수와 JS heap
- FPS와 event-to-paint 지연

### 예상 병목

코드상 예상 병목은 병합 시 전체 정렬·Map 순회, 표시 시 두 번째 전체 정렬, 전체 map과 누적 DOM이다. 실제 지배적 병목은 trace로 판정한다.

### 개선 후 After 측정 방법

동일 fixture와 동일 동작을 반복해 정렬된 병합 및 virtualization 적용 전후를 비교한다. DOM 감소만 확인하지 말고 스크롤 위치, 읽음 처리, 접근성, 첫 메시지 표시의 기능 회귀도 확인한다.

## 5.2 읽음 receipt 처리 테스트

### 테스트 대상

`applyReadReceipts`, `ReadStatus`, 메시지별 IntersectionObserver, Socket.IO `messagesRead` 처리다.

### 테스트 준비

메시지 수, 참여자 수, readers 수, 한 payload의 message ID 수가 다른 고정 시나리오를 만든다. 동일한 receipt event 순서와 중복 event를 재생할 수 있게 한다.

### Before 측정 방법

채팅 스크롤로 여러 메시지를 노출하고, 다른 사용자 receipt burst를 모의한다. Socket.IO 송수신 로그, React Profiler, Performance trace를 수집한다.

### 확인할 지표

- 초당 읽음 event 수와 payload 크기
- handler scripting time
- event당 render/commit 수와 duration
- observer 수, long task, FPS
- 읽음 표시 정확성과 중복 처리

### 예상 병목

코드상 `N×K` membership 검색과 메시지별 `P×R` unread 계산, 전체 messages 배열 참조 변경이 병목 후보이다.

### 개선 후 After 측정 방법

Set/index와 batching 적용 후 같은 event stream을 재생한다. CPU와 render 감소뿐 아니라 receipt 유실, 순서 역전, 읽음 표시 지연이 허용 범위인지 확인한다.

## 5.3 방 목록 Polling 테스트

### 테스트 대상

`ChatRoomsView`의 30초 interval, `refreshRooms`, `loadRooms`, `/api/rooms`, `RoomsTable` 렌더다.

### 테스트 준비

방 수가 고정된 계정과 여러 독립 브라우저 세션을 준비한다. 보이는 탭, 숨김 탭, visibility 복귀, 연결 끊김 조건을 구분한다.

### Before 측정 방법

최소 2분 이상 Network log를 보존해 요청 시각과 횟수를 기록한다. 동시 세션을 단계적으로 늘리며 서버 access log/APM의 RPS와 응답 시간을 함께 본다.

### 확인할 지표

- 세션당 `/api/rooms` 요청 횟수와 간격
- 응답 bytes와 duration
- 전체 RPS 및 서버 오류율
- 30초 갱신당 React render 횟수·duration
- 숨김 탭에서 요청이 실제 중단되는지

### 예상 병목

Socket.IO 증분 이벤트와 전체 목록 Polling의 중복 비용, 전체 payload 전송과 state 교체가 병목 후보이다. 백엔드 쿼리 비용은 별도 trace가 필요하다.

### 개선 후 After 측정 방법

같은 세션 수, 방 수, 관찰 시간으로 이벤트 기반 또는 변경분 조회 구조를 측정한다. 요청·bytes·RPS 감소와 함께 방 목록 정합성 및 visibility 복귀 후 최신 상태를 확인한다.

---

## 6. 백엔드 문제와 연결되는 부분

현재 프론트 코드로 확정할 수 있는 것은 endpoint와 호출 빈도·순서까지다. DB 쿼리 형태, N+1, MongoDB 부하는 백엔드 코드나 APM 없이는 확정할 수 없다.

### 방 목록 Polling 연결

보이는 목록 탭의 30초 interval
→ `attemptConnection()`
→ `GET /api/rooms` 전체 조회
→ 동시 목록 사용자 수에 비례한 요청 증가
→ 백엔드 목록 직렬화 및 데이터 조회 반복
→ 목록 API 내부에 N+1 또는 비효율 쿼리가 있다면 비용이 곱해짐
→ DB·CPU·네트워크 부하 및 응답 지연 가능

여기서 “N+1이 있다”는 것은 이 프론트 코드로 확인되지 않는다. 백엔드 trace에서 요청당 쿼리 수를 확인해야 한다.

### 방 입장 연결

입장 클릭
→ `POST /api/rooms/:id/join`
→ route 이동
→ `GET /api/rooms/:id`
→ Socket.IO `joinRoom`
→ join 결과에 메시지가 없을 때 `fetchPreviousMessages`
→ 동시 입장이 많으면 멤버십·상세·소켓 처리의 동시 부하 증가

REST join과 Socket join이 서버에서 같은 멤버십 작업을 반복하는지는 추정할 수 없으므로 correlation ID로 확인해야 한다. 이미 가입된 사용자의 멱등 처리 비용도 함께 본다.

### 읽음 이벤트 연결

각 메시지의 50% 노출
→ `markMessagesAsRead([messageId])`
→ 서버 읽음 저장
→ `messagesRead` broadcast
→ 각 클라이언트가 전체 메시지 배열 탐색 및 재렌더
→ 참여자와 노출 메시지가 많으면 이벤트·broadcast·클라이언트 CPU가 함께 증가 가능

서버가 단건 쓰기를 batch하는지는 현재 코드로 알 수 없다. 클라이언트가 짧은 구간의 message IDs를 묶으면 이벤트 수를 줄일 수 있지만 제품의 읽음 실시간성 요구와 서버 API 지원을 확인해야 한다.

### 파일 업로드 연결

허용 파일 선택
→ preview Blob 유지
→ `POST /api/files/upload`
→ 업로드 완료 후 Socket.IO 파일 메시지 전송
→ 큰 파일 또는 동시 업로드 증가
→ 백엔드 네트워크·스토리지 처리 증가

프론트에는 이미지 10MB, PDF 20MB, 전체 50MB 검사가 있으나 서버에서도 동일하거나 더 엄격한 검증이 필요하다. 클라이언트 제한은 우회 가능하기 때문이다.

---

## 7. 추천 진행 순서

문제 발견
→ 현재 source와 실제 runtime event를 연결해 확인
→ “데이터 규모가 커지면 어떤 지표가 왜 나빠지는가”라는 가설 설정
→ 고정 fixture·브라우저·네트워크·서버 조건 정의
→ Before 측정 및 trace 저장
→ 가장 큰 scripting, render, layout, network 또는 server 병목 확인
→ 한 번에 한 가지 개선 적용
→ 동일 조건으로 After 측정
→ Before/After 지표와 기능 정확성 비교
→ 효과가 확인된 변경만 유지

실제 프로젝트에서는 먼저 메시지 렌더링 TOP 1 테스트로 브라우저 측 확장성을 확인하고, 동시에 방 목록 access log로 서버 증폭 정도를 측정하는 것이 좋다. 이후 읽음 event stream을 재현해 두 문제가 서로 증폭하는지 확인한다. 방 입장 구조 변경은 프론트와 백엔드의 책임 계약을 먼저 정리한 뒤 진행한다.

> 핵심: 성능 개선은 코드 모양만 보고 끝내지 않고 같은 조건의 Before/After로 증명해야 한다. 속도뿐 아니라 메시지 순서, 읽음 정확성, 방 목록 정합성, 재연결과 메모리 회수까지 함께 통과해야 실제 개선이다.
