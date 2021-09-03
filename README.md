# 인터넷 도서구매

## Table of contents
  - [서비스 시나리오](#서비스-시나리오)
  - [분석/설계](#분석설계)
  - [구현](#구현)
    - [DDD의 적용](#DDD(DomainDrivenDesign)의-적용)
    - [Saga](#Saga)
    - [비동기식 호출과 Eventual Consistency](#비동기식-호출과-Eventual-Consistency)
    - [CQRS](#CQRS)
    - [Correlation](#Correlation)
    - [동기식 호출](#동기식-호출)
    - [GateWay](#GateWay)
    - [서킷 브레이커](#서킷-브레이커)  
    - [Polyglot Persistent/Polyglot Programming](#Polyglot-Persistent--Polyglot-Programming)
  - [운영](#운영)
    - [Pipeline](#Pipeline)   
    - [HPA](#HPA-(Horizontal-Pod-Autoscaler))
    - [무정지 재배포](#무정지-재배포)
    - [Self-Healing](#Self-Healing-(Liveness-probe))
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
     - 도서비 결제가 완료 되어야만 주문 신청을 완료할 수 있다-> Sync 호출
    2. 장애격리
     - 배송관리 기능이 수행되지 않더라도 주문은 365일 24시간 받을 수 있어야 한다-> Async(event-driven),Eventual Consistency
     - 결제시스템이 과중되면 도서주문을 잠시동안 받지 않고, 잠시 후에  결제하도록 유도한다-> Circuit breaker
    3. 성능
     - 고객은 언제든지 주문/배송현황을 MyPage에서 확인할 수 있어야 한다-> CQRS

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
   - 도서비 결제가 완료 되어야만 주문 신청을 완료할 수 있다 (Req/Res 방식, Sync호출)
  2) 장애격리 (O)
   - 배송기능이 수행되지 않더라도 주문은 365일/24시간 받을 수 있어야 한다 (Pub/Sub 방식, Async호출, Eventual Consistency)
   - 결제시스템이 과중되면 도서주문을 잠시동안 받지 않고, 잠시 후에 결제하도록 유도한다 (Circuit breaker 적용)
  3) 성능 (O)
   - 고객이 수시로 주문/배송현황을 MyPage에서 확인할 수 있어야 한다 (CQRS 구현)
``` 

#### Hexagonal Architecture Diagram 도출
 ![image](https://user-images.githubusercontent.com/87114545/131242875-445b806b-bf91-42f4-956d-d84fdb51ca0d.png)


</br>
 
## 구현
- 분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 
  각 BC별로 대변되는 마이크로 서비스들을 Spring Boot와 Java로 구현하였다.</br>
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
![image](https://user-images.githubusercontent.com/87114545/131857119-c4e668f7-9d9e-48af-ad55-9ec0553b26db.png)

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
  데이터 접근 어댑터를 자동생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다. (OrderRepository.java)
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
    ![image](https://user-images.githubusercontent.com/87114545/131857282-9fcb1eb2-1ef1-48aa-b30d-d12c2b801c3a.png)

  - 주문 확인</br>
    : orderId = 1인 주문 생성 </br> 
    ![image](https://user-images.githubusercontent.com/87114545/131857318-96c24337-8872-45d4-9a3a-5012df49bfc7.png)</br>

  - 결제 승인 확인</br>
    : orderlId = 1에 대한 paymentId = 1인 payment의 status 확인 </br> 
    ![image](https://user-images.githubusercontent.com/87114545/131857362-e93838a7-daaf-49c1-b444-4c7fec9b87c2.png)</br> 

  - 배송 시작 확인</br>
    : paymentId = 1에 대한  delivery 조회 됨</br>
    ![image](https://user-images.githubusercontent.com/87114545/131857387-0744064a-3569-4455-9260-9958990578bb.png)</br>

  - 주문/배송 현황 확인(MyPage)</br> 
    ![image](https://user-images.githubusercontent.com/87114545/131857696-c42efbb8-a063-4be2-9e9c-7d53a0bda94b.png)</br>

  - 주문 취소 </br> 
    ![image](https://user-images.githubusercontent.com/87114545/131858066-e968453d-60c7-49b1-a3d8-73af4e54aa81.png)</br>

  - 주문 취소 확인 </br> 
    ![image](https://user-images.githubusercontent.com/87114545/131858439-0e8ac81d-3e37-4865-9f2c-3c7eb683835f.png)</br>

  - 결제 취소 확인 </br> 
    ![image](https://user-images.githubusercontent.com/87114545/131858467-b47bdd8d-f73a-47c6-9b53-db8d84b337e8.png)</br>

  - 배송 취소 확인 </br> 
    ![image](https://user-images.githubusercontent.com/87114545/131858493-63ba1a92-b742-4d1c-b3e2-2ed037d77a18.png)</br></br>


### 비동기식 호출과 Eventual Consistency
- OrderPlaced -> PaymentApproved -> DeliveryStarted 순서로 Event가 처리됨</br>
 ![image](https://user-images.githubusercontent.com/87114545/131859059-3404717e-4a10-4fef-af59-df076de23b00.png)</br></br>


### CQRS
  - My Page에서 주문/결제/배송 상태 확인 (CQRS)</br>    
  ![image](https://user-images.githubusercontent.com/87114545/131859567-d211b0fa-e8b0-4947-a6d2-51937e632683.png)</br></br>


### Correlation 
  - OrderCanceled 이벤트 발생</br> 
  ![image](https://user-images.githubusercontent.com/87114545/131859757-1cfb102d-1a11-4c0e-825d-746912963520.png)</br> 

  - CorrelationId로 찾아서 삭제</br> 
  ![image](https://user-images.githubusercontent.com/87114545/131860203-4129c1a1-1f49-4672-910a-61bf0951c011.png)</br></br>  


### 동기식 호출
- 구현 필수 요건 중 주문(Order)-> 결제(Payment)간 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다.
- 호출 프로토콜은 이미 앞서 Rest Repository에 의해 노출되어있는 REST 서비스를 FeignClient를 이용하여 호출하도록 한다.</br></br>
- Order서비스 내부의 payment서비스
```java
@FeignClient(name ="payment", url="${api.url.payment}")
public interface PaymentService {
    
    @RequestMapping(method = RequestMethod.POST, value = "/payments", consumes = "application/json")
    public void startPayment(Payment payment);
}
```
 - Order서비스의 application.yaml
```java
api:
  url:
    payment: http://localhost:8082
```
- 동기식 호출 후 payment서비스 처리결과</br> 
  ![image](https://user-images.githubusercontent.com/87114545/131860656-f1a18f00-85cd-435b-8e74-006f1c28fb7b.png)</br></br>


### GateWay
  - API GateWay를 통하여 마이크로 서비스들의 진입점을 통일할 수 있도록 구현하였다. </br>
    - 서비스 직접 조회</br>
   ![image](https://user-images.githubusercontent.com/87114545/131860706-507c3413-04b5-4f03-8e65-c79d1bd50cbf.png)</br>
   
    - Gateway를 경유해서 조회</br>
   ![image](https://user-images.githubusercontent.com/87114545/131860725-1f310327-0fd2-4d9c-a620-e7fae10ab2aa.png)</br></br>


### 서킷 브레이커
- 서킷 브레이킹 프레임워크의 선택 (FeignClient + hystrix) </br>
- 요청처리 쓰레드에서 임계치 (610 Milliseconds) 이내로 Response가 내려오지 않으면</br>
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

### Polyglot Persistent / Polyglot Programming
- Polyglot Persistent 조건을 만족하기 위해 기존 h2 DB를 hsqldb로 변경하여 동작시킨다. (Order서비스의 pom.xml)</br>
![image](https://user-images.githubusercontent.com/87114545/131254432-a3072141-ecba-466a-977c-c88acef2bf76.png)


- pom.xml 파일 내 DB 정보 변경 및 재기동</br>
 ![image](https://user-images.githubusercontent.com/87114545/131755289-8d0fbbd5-e297-470a-8b0b-5c0d8e2664c4.png)</br>
- OrderPlaced 정상 처리</br>
 ![image](https://user-images.githubusercontent.com/87114545/131864258-55afcdeb-757f-4d72-b24f-1e64c62c3310.png)</br></br>


## 운영
### Pipeline
- 각 구현체들은 각자의 source repository 에 구성되었고, 사용한 CI/CD 플랫폼은 aws codebuild를 사용하였으며,</br>
  pipeline build script 는 각 프로젝트 폴더 이하에 buildspec.yml 에 포함되었다.</br></br>
 ![image](https://user-images.githubusercontent.com/87114545/131862693-11d97b61-9946-43f4-b617-9fbe7c7b1fe3.png)</br>


### HPA (Horizontal Pod Autoscaler)
- 자동화된 확장 기능 적용을 위해 order서비스 buildspec.yml에 resource 설정 추가 (CPU 최대/최소값 설정)</br>
```java
          resources:
            limits:
              cpu: 500m 
            requests:
              cpu: 200m
```

- 오토스케일링 설정 (해당 deployment 컨테이너의 최대값/최소값 설정)</br>
 ![image](https://user-images.githubusercontent.com/87114545/131937138-cfe201b7-fed2-49ae-9fdb-93fda3a4044a.png)</br>

- 워크로드를 100명, 30초 동안 부하를 걸어준다.</br>
 ![image](https://user-images.githubusercontent.com/87114545/131938005-e9ea92a9-3c8d-4f31-97dd-25f4d6b2edba.png)</br>

- 부하 테스트 수행전 </br>
 ![image](https://user-images.githubusercontent.com/87114545/131937421-cf8a9abc-648a-42ab-bac5-fc88d935db6e.png)</br>

- 일정시간 경과후 스케일 아웃 확인 (payment Pod 증가)</br>
 ![image](https://user-images.githubusercontent.com/87114545/131937637-ae49e4ea-a323-45bb-b6e5-8281f0ad8c43.png)</br>
 

### 무정지 재배포 (Readiness Probe)
- readiness 설정이 있는 payment v1 상태에서 siege 실행 (siege -c10 -t600S -v http://user5-payment)</br> 
- readiness 설정 주석 처리후 v2 배포</br> 
```java    
      #readinessProbe:
      #  httpGet:
      #    path: /actuator/health
      #    port: 8080
      #  initialDelaySeconds: 10
      #  timeoutSeconds: 2
      #  periodSeconds: 5
      #  failureThreshold: 10
```
- siege 실행 중 readiness가 없는 v2 배포 완료 후 availability 하락 확인</br>
  ![image](https://user-images.githubusercontent.com/87114545/131941708-8a5c72ab-097d-4e0b-9ace-8e80964d71b2.png) </br> 

- readiness 설정 후 v3 배포</br> 
- readiness 옵션 존재 시 동일 조건 부하에서 Availablility 100% 확인</br>
 ![image](https://user-images.githubusercontent.com/87114545/131944065-02497158-1ce8-4798-adec-11f35cc36054.png)</br>

 
 
### Self-Healing (Liveness probe)
- Liveness probe를 통해 Pod의 상태를 체크하다가, Pod의 상태가 비정상인경우 재시작한다. 
- Liveness probe 관련 설정 (payment서비스의 buildspec.yml) </br>
  /tmp/test 파일이 존재하는지 주기적 확인 후 비정상(파일 미존재) 판단시 자동으로 재시작)</br>
    
```java    
    containers:
      - name: $_PROJECT_NAME
        image: $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$_PROJECT_NAME:$CODEBUILD_RESOLVED_SOURCE_VERSION
        ports:
          - containerPort: 8080
        args:           
          - /bin/sh     
          - -c          
          - touch /tmp/test; sleep 30; rm -rf /tmp/healthy; sleep 600
        ...  
        ...  
        livenessProbe:
          exec: 
            command: 
            - cat    
            - /tmp/test
          initialDelaySeconds: 5
          periodSeconds: 5
```
- 해당 pod의 재시작 횟수 증가 확인</br>
    ![image](https://user-images.githubusercontent.com/87114545/131945238-36c24956-baea-4aea-924c-c95214d670c9.png)</br>

### ConfigMap
 - 변경 가능성이 있는 설정을 ConfigMap을 사용하여 관리
 - order서비스에서 호출하는 payment서비스 url 일부분을 ConfigMap 사용하여 구현 (order서비스의 application.yml)</br>
```java
    # api:
    #   url:
    #     payment: http://user15-payment:8080
    
    api:
      url:
        payment: ${config_url}
```
 - ConfigMap 정의 (Order서비스의 buildspec.yml)</br>
```java 
     env:
       - name: bookstore-cm
         valueFrom:          
           configMapKeyRef:  
             name: bookstore-cm
             key: config_url     
```
 - configMap 생성 및 확인 </br>
 ![image](https://user-images.githubusercontent.com/87114545/131945794-9a32367a-618b-4b7f-9935-441d64291e2a.png)</br>
 
 - 생성 후 서비스 정상 수행 확인 => 서비스 비정상 동작으로 확인 못함..</br>
