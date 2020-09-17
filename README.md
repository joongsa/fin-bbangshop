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







구현
분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 808n 이다)

cd reservation
mvn spring-boot:run

cd assignment
mvn spring-boot:run 

cd product
mvn spring-boot:run  

cd gateway
mvn spring-boot:run  

## DDD 의 적용
- 각 서비스내에 도출된 핵심 객체를 Entity 로 선언
  - 예약 -> reservation
  - 배정 -> assignment
  - 상품 -> product

## 적용 후 REST API 의 테스트
- 서비스의 예약처리 : http POST localhost:8088/reservations productId=1
- 서비스의 예약취소 처리 : http PATCH localhost:8088/reservations/1 status="예약취소"
- 예약상태 확인 : http localhost:8088/reservations/1
SAGA 패턴
취소에 따른 보상 트랜잭션 설계
예약 도메인에 취소 요청이 들어오면, 취소 이벤트가 발생하며 (ReservationCancelRequested) 배정 도메인의 취소 정책(ReservationCancel)을 호출한다. 배정의 예약이 삭제되고, 예약 취소 이벤트(ReservationCanceled)가 발생하며 상품 도메인 쪽의 수량 변경 정책(QuantityChange)을 호출하여 수량을 원복해 줌과 동시에 예약 도메인의 상태 변경 정책(statusChange)을 호출하여 정상적으로 예약이 취소 되었음을 알려준다.

취소 트랜잭션
다운로드11

CQRS
하나 이상의 데이터 소스에서 데이터를 프로젝션하는 아키텍처
고객은 자신의 예약 상태를 뷰 (OrderPage) 를 통해 확인할 수 있다. 예약 요청 이벤트(ReservationRequested)가 발생되면 자동으로 생성되며, 예약 접수 이벤트(ReservationAccepted)나 예약 취소 이벤트(ReservationCanced) 발생 시 예약 상태값이 변경된다.

OrderPage 뷰
다운로드22

동기식 호출
예약과 재고확인/재고변경 호출은 동기식 트랜잭션으로 처리

    /**
     * 예약접수를 신청하면 상품수량을 확인하고 예약가능/불가여부를 판단
     * */
    @PrePersist
    public void onPrePersist(){
        //Tshop.external.Product product = new Tshop.external.Product();

        String checkQuantity = ReservationApplication.applicationContext.getBean(Tshop.external.ProductService.class).checkProductQuantity(this.getProductId().toString());

        if(Integer.parseInt(checkQuantity) > 0){
            this.setStatus("예약신청");
        }else{
            this.setStatus("예약불가");
        }
    }
    
    /**
     * FeignClient를 사용한 ProductService 동기식호출
     **/
    @FeignClient(name="Product", url="${api.url.product}")
    public interface ProductService {

    @RequestMapping(method= RequestMethod.GET, path="/products/check",produces = "application/json")   
    @ResponseBody String checkProductQuantity(@RequestParam("productId") String productId);

}
예약신청을 하면 현 재고상태를 확이하여, 신청/불가 판단
@RequiredArgsConstructor
@Service
public class ProductService {
    private final ProductRepository productRepository;

    @Transactional
    public String checkQuantityByProductId(String productId) {

        Optional<Product> optionalProduct = productRepository.findById(Long.parseLong(productId));
        Product product = optionalProduct.orElseGet(Product::new);
        // 상품이 없을경우 재고0으로 전달
        if(product.getQuantity() == null) product.setQuantity(0);
        // 상품재고가 있는 경우 재고 -1 하고 할당으로 이벤트 전달
        if( product.getQuantity() > 0 ){
            product.setQuantity(product.getQuantity()-1);
            productRepository.save(product);
            //product.pulishQuantityChecked();
        }
        return product.getQuantity().toString() ;
    }
}
상품재고를 판단하여 리턴
    /**
     * 예약신청 가능이면 배정관리서비스로 예약번호 전송
     * */
    @PostPersist
    public void onPostPersist(){
        ReservationRequested reservationRequested = new ReservationRequested();
        BeanUtils.copyProperties(this, reservationRequested);
        if("예약신청".equals(this.getStatus())) reservationRequested.publishAfterCommit();
    }
    /**
상태가 예약신청가능한 상태면 배정
폴리글랏
고객관리 서비스(customercenter)팀에서는 DataBase를 H2가 아닌 비슷한 계열의 인메모리 DB hsql를 적용함

<name>customercenter</name>
<!-- <dependency>
<groupId>com.h2database</groupId>
<artifactId>h2</artifactId>
<scope>runtime</scope>
</dependency>-->

<dependency>
<groupId>org.hsqldb</groupId>
<artifactId>hsqldb</artifactId>
<version>2.4.0</version>
<scope>runtime</scope>
</dependency>
운영
CI/CD 설정
각 구현체들은 각자의 source repository 에 구성되었고, Deployment.yaml 설정을 통해 배포 됨

서킷 브레이킹
부하테스터 siege 툴을 통한 서킷 브레이커 동작 확인:
동시사용자 2명
5초 동안 실시
캡처1111

오토스케일 아웃
앞서 CB 는 시스템을 안정되게 운영할 수 있게 해줬지만 사용자의 요청을 100% 받아들여주지 못했기 때문에 이에 대한 보완책으로 자동화된 확장 기능을 적용하고자 한다.

상품서비스에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 10프로를 넘어서면 replica 를 10개까지 늘려준다:
kubectl autoscale deploy reservation --min=1 --max=10 --cpu-percent=10
CB 에서 했던 방식대로 워크로드를 30초 동안 걸어준다.
siege -c10 -t30S  -v --content-type "application/json" 'http://reservation:8080/reservations POST {"productId":1}'
오토스케일이 어떻게 되고 있는지 모니터링을 걸어둔다:
kubectl get deploy product -w
어느정도 시간이 흐른 후 (약 30초) 스케일 아웃이 벌어지는 것을 확인할 수 있다:
캡처4111

siege 의 로그를 확인
캡처4112
