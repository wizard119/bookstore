# 인터넷 도서구매

## Table of contents
  - [서비스 시나리오](#서비스-시나리오)
  - [분석/설계](#분석설계)
  - [구현](#구현)
    - [DDD의 적용](#DDD의-적용)
    - [Saga](#Saga)
    - [비동기식 호출과 Eventual Consistency](#비동기식-호출과-Eventual-Consistency)
    - [CQRS](#CQRS)
    - [Correlation](#Correlation)
    - [동기식 호출](#동기식-호출)
    - [Gateway](#Gateway)
    - [서킷 브레이킹](#서킷-브레이킹)  
    - [Polyglot Persistent/Polyglot Programmin](#Polyglot-Persistent-/-Polyglot-Programming)
  - [운영](#운영)
    - [Deploy/Pipeline 설정](#Deploy,-Pipeline-설정)   
    - [HPA](#AutoScale-Out)
    - [무정지 재배포](#무정지-재배포)
    - [Self Healing](#Self-Healing)
    - [ConfigMap](#ConfigMap)

## 서비스 시나리오
    [기능적 요구사항]
        1. 고객이 도서메뉴를 선택하여 주문한다
        2. 고객은 주문한 도서의 구매비용을 결제한다
        3. 결제가 완료되면 도서배송을 시작한다
        4. 고객은 도서주문을 취소 할 수 있다
        5. 도서주문이 취소되면 결제가 취소된다
        6. 결제가 취소되면 배송이 취소된다
        7. 고객은 언제든지 주문/배송현황을 조회한다 (CQRS-View) 

    [비기능적 요구사항]
        1. 트랜잭션 
          - 도서비 결제가 완료 되어야만 주문 신청을 완료할 수 있다 -> Sync 호출
        2. 장애격리
          - 배송관리 기능이 수행되지 않더라도 주문은 365일 24시간 받을 수 있어야 한다 -> Async(event-driven),Eventual Consistency
          - 결제시스템이 과중되면 도서주문을 잠시동안 받지 않고, 잠시 후에  결제하도록 유도한다 -> Circuit breaker,fallback
        3. 성능
          - 고객은 언제든지 주문/배송현황을 MyPage에서 확인할 수 있어야 한다 -> CQRS

## 분석/설계
### Event Storming 결과
    MSAEz 로 모델링한 이벤트스토밍 결과
    http://www.msaez.io/#/storming/MIHRQx5rkxPj1aZzp4LcWOunLAQ2/8468ff25a224bcbb48aa10e01ac833e3
    
 ![image](https://user-images.githubusercontent.com/87114545/131238889-39de0c08-2ece-496a-aeea-5ae7399e434d.png)

    

#### 완성된 모형
 ![image](https://user-images.githubusercontent.com/87114545/131238869-98960f77-6d37-4000-bb05-45f77c18d9e9.png)</br>
#### 1. 기능적 요구사항 검증
 ![image](https://user-images.githubusercontent.com/87114545/131314017-ed56dd00-8e01-4c7f-97d2-4d931a1a270a.png)
```   
  1) 고객이 도서메뉴를 선택하여 주문하고, 구매비용을 결제하면 도서배송을 시작한다 (O)
  2) 고객이 주문을 취소하면, 결제가 취소되고, 배송이 취소된다 (O)
  3) 고객은 언제든지 주문/배송현황을 조회한다 (O)
```  
#### 2. 비기능적 요구사항 검증
 ![image](https://user-images.githubusercontent.com/87114545/131309204-79249468-e647-42bd-a9da-1cd327b4da0c.png)
```   
  1) 트랜잭션 (O)
   - 도서비 결제가 완료 되어야만 주문 신청을 완료할 수 있다 (Req/Res 방식, Sync 호출)
  2) 장애격리 (O)
   - 배송기능이 수행되지 않더라도 주문은 365일/24시간 받을 수 있어야 한다 (Pub/Sub 방식, Async 호출, Eventual Consistency)
   - 결제시스템이 과중되면 도서주문을 잠시동안 받지 않고, 잠시 후에 결제하도록 유도한다 (Circuit breaker 적용)
  3) 성능 (O)
   - 고객이 수시로 주문/배송현황을 MyPage에서 확인할 수 있어야 한다 (CQRS 구현)
``` 

#### Hexagonal Architecture Diagram 도출
 ![image](https://user-images.githubusercontent.com/87114545/131242875-445b806b-bf91-42f4-956d-d84fdb51ca0d.png)


</br>
 
## 구현
- 분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 Spring Boot와 Java로 구현하였다.</br>
 (각 마이크로 서비스의 포트 넘버는 8081 ~ 808n 이다)</br>
```
cd order
mvn spring-boot:run

cd payment
mvn spring-boot:run 

cd delivery
mvn spring-boot:run

cd mypage
mvn spring-boot:run

cd gateway
mvn spring-boot:run

```
![image](https://user-images.githubusercontent.com/87114545/131244348-352a7bc4-3f1a-4865-99aa-466a85aed7b2.png)


- AWS 클라우드의 EKS 서비스 내에 마이크로 서비스를 모두 빌드한다
![image](https://user-images.githubusercontent.com/87048633/130029192-6520c94a-ffe2-4bc3-93c9-3f3d6498bfe1.png)
![image](https://user-images.githubusercontent.com/87048633/130029296-b2324bb8-08de-4749-ae77-8e4a9de4cfc5.png)

### DDD(Domain-Driven-Design)의 적용
- 각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다. (Order.java)</br>
```java
package com.example.order;

import javax.persistence.*;

import com.example.order.external.Payment;
import com.example.order.external.PaymentService;

import org.springframework.beans.BeanUtils;
import java.util.List;
import java.util.Date;

@Entity
@Table(name="order_table")
public class Order {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private Long customerId;
    private Long productId;
    private String address;
    private String status;
    private int amt;

    @PostPersist
    public void onPostPersist(){
        OrderPlaced orderPlaced = new OrderPlaced();
        BeanUtils.copyProperties(this, orderPlaced);
        orderPlaced.publishAfterCommit();
    }

    @PreRemove
    public void onPreRemove(){
        OrderCanceled orderCanceled = new OrderCanceled();
        BeanUtils.copyProperties(this, orderCanceled);
        orderCanceled.publishAfterCommit();
    }

    public Long getId() {
        return this.id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public Long getCustomerId() {
        return this.customerId;
    }

    public void setCustomerId(Long customerId) {
        this.customerId = customerId;
    }

    public Long getProductId() {
        return this.productId;
    }

    public void setProductId(Long productId) {
        this.productId = productId;
    }

    public String getAddress() {
        return this.address;
    }

    public void setAddress(String address) {
        this.address = address;
    }

    public String getStatus() {
        return this.status;
    }

    public void setStatus(String status) {
        this.status = status;
    }

    public int getAmt() {
        return this.amt;
    }

    public void setAmt(int amt) {
        this.amt = amt;
    }

}
```
- Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형에 대한 별도의 처리가 없도록 </br>
  데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다. (OrderRepository.java)
```java
package com.example.order;

import org.springframework.data.repository.CrudRepository;

public interface OrderRepository extends CrudRepository<Order, Long> {

}
```
</br>

### Saga
- 적용 후 REST API 의 테스트</br>

  - 주문 등록</br>
    ![image](https://user-images.githubusercontent.com/87048624/130166446-f4f4696d-719c-434b-8793-e41a801e051b.png)</br>

  - 주문 확인</br>
    : orderId = 1인 주문 생성 </br> 
    ![image](https://user-images.githubusercontent.com/87048624/130166720-4ecceef4-3974-4369-b731-ed5e233438aa.png)</br>

  - 결제 승인 확인</br>
    : orderlId = 1에 대한 paymentId = 1인 payment의 status 확인 </br> 
    ![image](https://user-images.githubusercontent.com/87048624/130166870-aba9ade8-9b45-4b68-998f-4cd942b3e415.png)</br> 

  - 배송 시작 확인</br>
    : paymentId = 1에 대한  delivery 조회 됨</br>
    ![image](https://user-images.githubusercontent.com/87048624/130167028-d6d5b306-a48d-40da-bf8f-a079b3cc0859.png)</br>

  - 주문 취소</br> 
    ![image](https://user-images.githubusercontent.com/87048624/130167484-d6733c1b-153a-4a55-a1ae-fd2b38209187.png)</br>
   
  - 렌탈 취소 확인</br>  
    ![image](https://user-images.githubusercontent.com/87048624/130167567-033529f9-6d59-428d-84c0-63974b32fae8.png)</br>

  - 결제 취소 확인</br>  
    ![image](https://user-images.githubusercontent.com/87048624/130167646-c01d6969-5728-4493-a681-354363bd8dc4.png)</br>

  - 배송 취소 확인</br>  
    ![image](https://user-images.githubusercontent.com/87048624/130167750-1d6ab2f7-d563-420d-af96-1cc62417a7dc.png)</br>


### 비동기식 호출과 Eventual Consistency
- OrderPlaced -> PaymentApproved -> DeliveryStarted 순서로 Event가 처리됨</br>
 ![image](https://user-images.githubusercontent.com/87048624/130095576-6a139cd6-67c3-4667-ab7d-a4a961114e10.png) </br>

- 결과적 일치 확인</br>
 ![image](https://user-images.githubusercontent.com/87048624/130171502-462a0563-cc65-428a-a22d-4be469e0fae9.png)</br>


### CQRS
  - My Page에서 렌탈 신청 여부/결제성공여부/배송상태확인 (CQRS)</br>    
    : mypage 조회시rentalPlaced 이벤트까지만 수신내역 확인, 모든 이벤트 수신내역 확인</br>      
     ![image](https://user-images.githubusercontent.com/87048624/130083651-18549595-ec82-4bca-a5e7-3d23952872dc.png)</br>


### Correlation 
  - RentalCanceled 이벤트 발생</br> 
  ![image](https://user-images.githubusercontent.com/87048624/130172126-56738d8c-2968-4728-a32c-88af68eb7416.png)</br> 

  - CorrelationId로 찾아서 삭제</br> 
  ![image](https://user-images.githubusercontent.com/87048624/130172215-6c23c8a6-a009-419c-873e-83ea556fd2c6.png)</br> 


### 동기식 호출
- 구현 필수 요건 중 주문(Order) -> 결제(Payment)간 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다.
- 호출 프로토콜은 이미 앞서 Rest Repository에 의해 노출되어있는 REST 서비스를 FeignClient를 이용하여 호출하도록 한다.</br></br>
- Order 서비스 내부의 payment 서비스
```java
@FeignClient(name ="payment", url="${api.url.payment}")
public interface PaymentService {
    
    @RequestMapping(method = RequestMethod.POST, value = "/payments", consumes = "application/json")
    public void startPayment(Payment payment);
}
```
 - Order 서비스의 application.yaml
```java
api:
  url:
    payment: http://localhost:8082
```
- 동기식 호출 후 payment 서비스 처리결과</br> 
  ![image](https://user-images.githubusercontent.com/87048624/130095444-62dfc940-cecf-4791-907e-bd0136ce1b25.png)</br>


### GateWay
  - API GateWay를 통하여 마이크로 서비스들의 진입점을 통일할 수 있도록 GateWay를 적용하였다. </br>
    - 서비스 직접 조회</br>
   ![image](https://user-images.githubusercontent.com/87048624/130100882-d053b088-2e49-476b-8521-fbe29367e739.png)</br>
   
    - Gateway로 조회</br>
   ![image](https://user-images.githubusercontent.com/87048624/130166073-6e213177-ceb8-44fa-9b12-8538bfd9b754.png)</br>


### 서킷 브레이커
- 서킷 브레이킹 프레임워크의 선택 (FeignClient + hystrix) </br>
- 요청처리 쓰레드에서 임계치 (610 Milliseconds) 이내로 Respons가 내려오지 않으면</br>
  서킷브레이커가 작동하도록 설정 (Order서비스의 application.yml)</br>
```java
 feign:
   hystrix:
     enabled: true

 hystrix:
   command:
     # 전역설정
     default:
       execution.isolation.thread.timeoutInMilliseconds: 610
```

- 피호출 서비스의 임의 부하 처리 : 400~620 밀리 사이에서 처리하도록 함 (Payment.java)</br>
```java
    @PostPersist
    public void onPostPersist(){

        // circuit breaker start
         try {
             Thread.currentThread().sleep((long) (400 + Math.random() * 220));
         } catch (Exception e) {
             //TODO: handle exception
             e.printStackTrace();
         }
         circuit breaker end

        PaymentApproved paymentApproved = new PaymentApproved();
        BeanUtils.copyProperties(this, paymentApproved);
        paymentApproved.publishAfterCommit();
    }
```    

- 서킷 브레이크 적용전

  ![image](https://user-images.githubusercontent.com/87048624/130067551-49d8804e-af47-4913-a727-27ec6d01945b.png)

- 서킷 브레이크 적용후 

  ![image](https://user-images.githubusercontent.com/87048624/130067583-be97a8b1-6080-4145-a681-f87b476f58d3.png)</br>


### Polyglot Persistent / Polyglot Programming
- Polyglot Persistent 조건을 만족하기 위해 기존 h2 DB를 hsqldb로 변경하여 동작시킨다. (Order 서비스의 pom.xml)</br>
![image](https://user-images.githubusercontent.com/87114545/131254432-a3072141-ecba-466a-977c-c88acef2bf76.png)


- pom.xml 파일 내 DB 정보 변경 및 재기동</br>
![image](https://user-images.githubusercontent.com/87048624/130165475-6efe2537-9941-4e0a-a689-a6de60a8ee67.png)</br>
- rental 서비스 재기동 후 DB에 데이터 없는 상태</br>
 ![image](https://user-images.githubusercontent.com/87048624/130172977-b3fc6202-4f63-4345-9156-d6d579aab07d.png)</br>
- rentalPlaced 정상 처리</br>
 ![image](https://user-images.githubusercontent.com/87048624/130173106-ba3f9f19-7be4-4d9a-b192-7ba4fb433433.png)</br>
- Pub/sub 결과도 정상</br>
 ![image](https://user-images.githubusercontent.com/87048624/130173152-57344c97-1f09-4aa6-b133-dc7a1961816f.png)</br>




## 운영
### Deploy, Pipeline 설정
- 각 구현체들은 각자의 source repository 에 구성되었고, 사용한 CI/CD 플랫폼은 aws codebuild를 사용하였으며,</br>
  pipeline build script 는 각 프로젝트 폴더 이하에 buildspec.yml 에 포함되었다.</br>
  ![image](https://user-images.githubusercontent.com/87048633/130006493-f79b40dc-242d-4684-95eb-e4a305abb6ef.png)
  ![image](https://user-images.githubusercontent.com/87048633/130006762-19c4648c-0e27-461b-897f-59aeeddb2bc6.png)
  ![image](https://user-images.githubusercontent.com/87048633/130007397-522fdd2e-cd61-4364-86b4-276afff0248d.png)
  ![image](https://user-images.githubusercontent.com/87048633/130007020-ce217d04-844f-423b-8bae-b141bc377ec8.png)</br>
  


### HPA (Horizontal Pod Autoscaler)
- 자동화된 확장 기능 적용을 위해 주문 서비스 buildspec.yaml에 resource 설정 추가 (CPU 최대/최소값 설정)</br>
```java
          resources:
            limits:
              cpu: 500m 
            requests:
              cpu: 200m
```

- 오토스케일링 설정 (해당 deployment 컨테이너의 최대값/최소값 설정)
```
  kubectl autoscale deployment user5-rental --cpu-percent=50 --min=1 --max=10   
``` 
- 워크로드를 110명, 30초 동안 부하를 걸어준다.</br>
 ![image](https://user-images.githubusercontent.com/87048624/130164937-5be87dce-e3dd-4f3e-b1cd-aa02610c13de.png)</br>
 
- AutoScale이 어떻게 되고 있는지 모니터링을 걸어둔다.</br>
 ![image](https://user-images.githubusercontent.com/87048633/130032165-431e8c28-1d82-4a9a-9391-ddda9cae46a7.png)</br>
- 어느정도 시간이 흐른 후 (약 30초) 스케일 아웃이 벌어지는 것을 확인할 수 있다. </br>
 ![image](https://user-images.githubusercontent.com/87048633/130032517-f53b5f92-acf0-4e68-908e-f60cdc2044ab.png)</br>
 ![image](https://user-images.githubusercontent.com/87048633/130032462-2dba6d76-6bcf-41f2-a391-936f9aa1e5d0.png)</br>
- siege 의 로그를 보아도 전체적인 성공률이 높아진 것을 확인 할 수 있다.</br>
 ![image](https://user-images.githubusercontent.com/87048633/130032321-6b55498c-56d0-40e2-8b46-c83aa66cb951.png)</br>


### 무정지 재배포
- readiness probe 를 통해 이후 서비스가 활성 상태가 되면 유입을 진행시킨다.</br> 
- 주석처리하여 v2 배포</br> 
 ![image](https://user-images.githubusercontent.com/87048624/130180233-576fdee7-ab33-488f-a092-7bd29876b542.png)</br>
- siege -c10 -t150S -v http://user5-rental 부하 후 readiness 주석 삭제 후 배포 하여 Availablility 100% 이하 확인</br>
 ![image](https://user-images.githubusercontent.com/87048624/130177311-f7a5638a-2f5a-43e9-8a4e-c54aeb9d7ed8.png) </br> 
- 주석해제 후 v3 배포</br> 
 ![image](https://user-images.githubusercontent.com/87048624/130177611-4330cb64-4ddb-4dad-9373-dc5fa9c257f3.png)</br>
- siege -c10 -t150S -v http://user5-rental 부하(동일조건)  readiness 옵션 존재 시 Availablility 100% 확인</br>
 ![image](https://user-images.githubusercontent.com/87048624/130177836-c04e4900-96c3-4079-993f-da53dce775c2.png)</br>

 
 
### Self-Healing (Liveness probe)
- Liveness probe 를 통해 Pod의 상태를 체크하다가, Pod의 상태가 비정상인경우 재시작한다. 
- Order 서비스 yaml파일에 Liveness probe 설정을 한다.</br>
  /tmp/healthy 파일이 존재하는지 주기적 확인 후 비정상(파일 미존재) 판단시 자동으로 재시작)</br>
    
```java    
    containers:
      - name: $_PROJECT_NAME
        image: $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$_PROJECT_NAME:$CODEBUILD_RESOLVED_SOURCE_VERSION
        ports:
          - containerPort: 8080
        args:           
          - /bin/sh     
          - -c          
          - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600
        ...  
        ...  
        livenessProbe:
          exec: 
            command: 
            - cat    
            - /tmp/healthy
          initialDelaySeconds: 5
          periodSeconds: 5
```

  - 해당 pod가 재시작하는걸 확인한다.   
    ![image](https://user-images.githubusercontent.com/87048624/130086524-d479788b-1023-4a95-906d-dad1a9cef1e5.png) 
    ![image](https://user-images.githubusercontent.com/87048624/130086552-2c59944d-6f71-4332-8c82-1ce5eff50a85.png)</br>

### ConfigMap 
 - 변경 가능성이 있는 설정을 ConfigMap을 사용하여 관리
 - order 서비스에서 호출하는 payment 서비스 url 일부분을 ConfigMap 사용하여 구현 (order서비스의 application.yml)</br>
```java
#api:
#  url:
#    payment: http://localhost:8082

api:
  url:
    payment: ${configurl}
```
 - ConfigMap 정의 (Order서비스의 Buildspec.yml)</br>
```java 
             env:
               - name: port-list     #configmap
                 valueFrom:          #configmap
                   configMapKeyRef:  #configmap
                     name: port-list #configmap
                     key: port2      #configmap
```

 - 생성전 배포후 rental pod 수행안함 </br>
  ![image](https://user-images.githubusercontent.com/87048624/130183020-1a521507-f678-4ba3-b0ee-4d21c0edeeed.png)</br>
 - configMap 생성
  ```
   kubectl create configmap bookstore_cm --from-literal=config_url=http://payment:8082
  ``` 
 - order pod 수행</br>
  ![image](https://user-images.githubusercontent.com/87048624/130183035-b4872b21-8678-4757-84b4-416e1736f19e.png)</br>

