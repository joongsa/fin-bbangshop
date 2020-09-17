# 한정판 빵집 키오스크 : bbangshop

## 서비스 소스 레파지토리
- https://github.com/myjukey/fin-reservation.git
- https://github.com/myjukey/fin-assignment.git
- https://github.com/myjukey/fin-bread.git
- https://github.com/myjukey/fin-page.git
- https://github.com/myjukey/fin-gateway.git

## 서비스 시나리오

### 기능적 요구사항
1. 고객이 빵집 키오스크에서 예약을 한다. 
2. 한정판 빵의 재고 여부를 확인하여 구입가능/불가능 여부를 알려준다.
3. 구입가능 시 빵 재고가 감소하고 자동으로 주문정보가 배정시스템으로 전달된다.
4. 배정시스템에서는 최적의 제빵사를 배정한다.( 랜던함수 )
5. 제빵사가 배정되면  [breadSucceed] 상태를 예약관리로 보낸다. 
6. 고객은 빵 수령 이전에 예약주문을 취소할 수 있다.
7. 예약취소가 접수 [cancellation]되면 배정된 주문데이터를 삭제하고, 해당 빵 재고를 원복한다.
8. 정상 취소처리가 되면 [breadCanceled] 상태가 된다.

### 비기능적 요구사항

#### 트랜잭션
 - 빵 재고가 없는 경우 예약은 접수처리가 되지 않는다. Sync 호출(Req/Res)

#### 장애격리
 - 배정관리 서비스가 되지않더라도 예약접수는 정상적으로 처리가 되어야한다. Async (event-driven)

## 이벤트스토밍

<img src="https://user-images.githubusercontent.com/68719151/93406808-93351300-f8cb-11ea-9a13-ea70247a6042.JPG" width="90%"></img>

## 구현
- 분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트로 구현
- 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 7071 ~ 707n 이다)

### DDD 의 적용
- 각 서비스내에 도출된 핵심 객체를 Entity 로 선언
  - 예약 -> reservation
  - 배정 -> assignment
  - 빵   -> bread

### 적용 후 REST API 의 테스트

- 한정판 빵 수량 등록 : http POST http://localhost:7073/breads breadName=cookie quantity=100
- 빵 예약 : http POST http://localhost:7071/reservations breadId=2
- 빵 예약 취소 : http PATCH http://localhost:7071/reservations/1 status=cancellation
- 예약확인 : http GET http://localhost:7071/reservations
- 배정확인 : http GET http://localhost:7072/assignments
- 빵 재고확인 : http GET http://localhost:7073/breads
- 뷰 확인 : http GET http://localhost:7074/pages

## SAGA 패턴

- 예약을 하면, 재고가 감소하고 배정이 되며 예약상태값이 breadSucceed로 변경된다. 
- 예약을 취소하면 재고가 원복되고 배정은 삭제되며 예약상태값이 breadCanceled로 변경된다. 

<img src="https://user-images.githubusercontent.com/68719151/93407728-caa4bf00-f8cd-11ea-816d-440d78b99fc2.JPG" width="90%"></img>

<img src="https://user-images.githubusercontent.com/68719151/93407734-cc6e8280-f8cd-11ea-9fb2-a57fdcc2efeb.JPG" width="90%"></img>

<img src="https://user-images.githubusercontent.com/68719151/93407740-ce384600-f8cd-11ea-9949-bd8d5f0e603b.JPG" width="90%"></img>

<img src="https://user-images.githubusercontent.com/68719151/93407745-d0020980-f8cd-11ea-8750-bc9476cfe069.JPG" width="90%"></img>

<img src="https://user-images.githubusercontent.com/68719151/93407749-d1cbcd00-f8cd-11ea-975d-9890415029f9.JPG" width="90%"></img>

<img src="https://user-images.githubusercontent.com/68719151/93407750-d2fcfa00-f8cd-11ea-8b7e-308a9a13b3f4.JPG" width="90%"></img>

<img src="https://user-images.githubusercontent.com/68719151/93407753-d42e2700-f8cd-11ea-8090-7c92f062303c.JPG" width="90%"></img>

<img src="https://user-images.githubusercontent.com/68719151/93407761-d7c1ae00-f8cd-11ea-962e-a044f7a501ec.JPG" width="90%"></img>

<img src="https://user-images.githubusercontent.com/68719151/93407767-dabc9e80-f8cd-11ea-892a-43b35b790ec1.JPG" width="90%"></img>

## CQRS

- 고객은 자신의 예약 상태를 뷰(Page) 를 통해 확인할 수 있다. 
<img src="https://user-images.githubusercontent.com/68719151/93408111-b90fe700-f8ce-11ea-8ba6-a39abf52f577.JPG"></img>


## 동기식 호출 

- 예약 시 재고확인하는 부분을 FeignClient를 사용하여 동기식 트랜잭션으로 처리 

<img src="https://user-images.githubusercontent.com/68719151/93408247-08eeae00-f8cf-11ea-97d3-3e857b4deac7.JPG"></img>

<img src="https://user-images.githubusercontent.com/68719151/93408251-0ab87180-f8cf-11ea-849b-6b3c584f5bd2.JPG"></img>


## 폴리글랏

- view페이지인 page 서비스에서는 DB hsql를 적용함

<img src="https://user-images.githubusercontent.com/68719151/93408246-0724ea80-f8cf-11ea-81c5-17fd29f96ba5.JPG"></img>

## 서킷 브레이킹

 - istio를 이용한 CB TEST 

<img src="https://user-images.githubusercontent.com/68719151/93408579-e3ae6f80-f8cf-11ea-8459-ac5f7ef7cd08.JPG"></img>

<img src="https://user-images.githubusercontent.com/68719151/93408602-ec06aa80-f8cf-11ea-8ee6-a48d9c01a7d7.JPG"></img>

<img src="https://user-images.githubusercontent.com/68719151/93408584-e5783300-f8cf-11ea-9bf1-982a5e87ebc5.JPG"></img>

<img src="https://user-images.githubusercontent.com/68719151/93408587-e6a96000-f8cf-11ea-8834-f284425f45ff.JPG"></img>


## 오토스케일 아웃

 - Mertric 서버 설치 후 오토스케일링 TEST
 - reservation의 deployment.yaml수정 후 배포 

<img src="https://user-images.githubusercontent.com/68719151/93408895-9f6f9f00-f8d0-11ea-9c2c-e40e4335e90b.JPG"></img>

<img src="https://user-images.githubusercontent.com/68719151/93408910-aac2ca80-f8d0-11ea-90ff-75d3c01f52b8.JPG"></img>

<img src="https://user-images.githubusercontent.com/68719151/93408953-c1692180-f8d0-11ea-9ef0-dde5cf3e84e4.JPG"></img>

<img src="https://user-images.githubusercontent.com/68719151/93408929-b4e4c900-f8d0-11ea-8d57-8013d1d2840a.JPG"></img>

<img src="https://user-images.githubusercontent.com/68719151/93408943-bc0bd700-f8d0-11ea-9fa9-44cea466bf4b.JPG"></img>

<img src="https://user-images.githubusercontent.com/68719151/93408956-c332e500-f8d0-11ea-9b59-cd46fe189dd2.JPG"></img>

