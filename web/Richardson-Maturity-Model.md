# Richardson Maturity Model
- 마틴 파울러가 소개한 개념으로 REST API가 얼마나 성숙한지 평가하기 위한 척도
- RESTful 원칙을 얼마나 잘 준수하고 있는지 4단계로 구분함
### 레벨 0: The swamp of POX
- 단일 URI를 사용해서 모든 작업을 POST 요청으로 처리하는 방식
- 작업의 종류는 요청 본문(body)에 의해서 결정
```shell
POST /bookingService HTTP/1.1
Content-Type: application/json
{
 "action": "retrieveDestinations"
}
```
- /bookingService 라는 엔드포인트로 POST 요청
- action에 명시되어 있는 작업을 수행
- 하나의 엔드포인트로 모든 작업을 수행
### 레벨1: Resource
- 리소스를 통해 개별 엔드포인트로 분리
- 여전히 모든 요청은 POST
- 리소스를 식별할 수 있지만 아직 RESTful하지 않음
```shell
POST /bookingDestinations
POST /bookingReservations
POST /bookingRooms
POST /bookingFeedback
```
- 레벨 2로 발전하기 위해서는 HTTP Method를 활용하여 리소스에 대한 다양한 작업을 수행할 수 있어야함
### 레벨2:  HTTP verbs
- HTTP Method(GET, POST, PUT, DELETE 등)를 사용하여 리소스를 더 구체적으로 분리
```shell
GET /books
POST /books
PUT /books/{id}
DELETE /books/{id}
```
### 레벨3: Hypermedia controls
- Hypermedia as the engine of application state(HATEOAS)를 도입하여 `response`에 포함된 링크를 통해 클라이언트가 추가 작업을 수행할 수 있도록 함
- 이를 통해 클라이언트는 서버의 전체 구조를 알 필요 없이 서버가 제공해주는 리소스와 링크만으로도 필요한 작업을 수행하게 됨
- WIKI: [HATEOAS](https://en.wikipedia.org/wiki/HATEOAS)
```shell
GET /reservations/12345 HTTP/1.1
```
`response`
```json
 "reservationId": "12345",
 "hotel": "Astoria",
 "checkin": "2024-07-01",
 "checkout": "2024-07-10",
 "status": "confirmed",
 "_links": {
  "self": {
   "href": "/reservations/12345"
  },
  "cancel": {
   "href": "/reservations/12345/cancel",
   "method": "POST"
  },
  "update": {
   "href": "/reservations/12345/update",
   "method": "PUT"
  }
 }
}
```
- `response`에 포함된 `_link` 를 통해 클라이언트가 필요한 작업을 작업할 수 있음
- 클라이언트는 API의 사전 지식 없이 서버가 제공해주는 링크를 통해 필요한 작업을 수행할 수 있음
- 상태 전이를 다루기위해 더 많은 로직이 필요할 수 있음 -> 복잡성 증가
- 추가적인 하이퍼 링크 데이터를 포함하므로 `response`의 크기가 증가할 수 있음
- 기존 시스템에서 도입하기에는 어려움 -> 초기 단계에서부터 고려해야 효과적