# 
# Musical Concert Booking
![image](https://user-images.githubusercontent.com/20183369/135119104-e58931a1-87e6-4b71-b3e1-5d4bbc38c996.png) ![image](https://user-images.githubusercontent.com/20183369/135119302-bc0ced91-77af-4334-89bc-242a1f9756be.png)

# 뮤지컬 콘서트티켓 예약시스템

본 예제는 MSA/DDD/Event Storming/EDA 를 포괄하는 분석/설계/구현/운영 전체 구현과제를 구성한 예제입니다.
이는 Cloud Native Application의 개발에 요구되는 체크포인트들을 통과하기 위한 예시 답안을 포함합니다.
- 체크포인트 : https://workflowy.com/s/assessment-check-po/T5YrzcMewfo4J6LW


# Table of contents

- [뮤지컬 콘서트티켓 예약](#---)
  - [서비스 시나리오](#서비스-시나리오)
  - [체크포인트](#체크포인트)
  - [분석/설계](#분석설계)
  - [구현:](#구현-)
    - [DDD 의 적용](#ddd-의-적용)
    - [폴리글랏 퍼시스턴스](#폴리글랏-퍼시스턴스)
    - [폴리글랏 프로그래밍](#폴리글랏-프로그래밍)
    - [동기식 호출 과 Fallback 처리](#동기식-호출-과-Fallback-처리)
    - [비동기식 호출 과 Eventual Consistency](#비동기식-호출-과-Eventual-Consistency)
  - [운영](#운영)
    - [CI/CD 설정](#cicd설정)
    - [동기식 호출 / 서킷 브레이킹 / 장애격리](#동기식-호출-서킷-브레이킹-장애격리)
    - [오토스케일 아웃](#오토스케일-아웃)
    - [무정지 재배포](#무정지-재배포)
    - [개발 운영 환경 분리](#개발-운영-환경-분리)
    - [모니터링](#모니터링)

# 서비스 시나리오


기능적 요구사항
1. 고객은 원하는 뮤지컬을 예약할 수 있다.
2. 고객은 예약한 뮤지컬의 티켓값을 결제한다.
3. 티켓이 결제되면 티켓 배송이 시작된다.
4. 티켓이 결제취소되면 티켓 배송이 취소된다.
5. 고객은 뮤지컬콘서트 예약 상황을 언제든지 볼 수 있다.

비기능적 요구사항
1. 트랜잭션
    1. 뮤지컬 콘서트티켓 결제가 완료 되어야만 뮤지컬 예약 완료 할 수 있음 Sync. 호출
    
2. 장애격리
    1. 뮤지컬티켓 예약 시스템이 과중되면 사용자를 잠시동안 받지 않고 예약을 잠시 후에 하도록 유도한다  Circuit breaker
    
3. 성능
    1. 고객은 마이페이지에서 예약한 뮤지컬과 티켓 배송 상태를 확인할 수 있어야 한다  CQRS

# 체크포인트

- 분석 설계
  - 이벤트스토밍: 
    - 스티커 색상별 객체의 의미를 제대로 이해하여 헥사고날 아키텍처와의 연계 설계에 적절히 반영하고 있는가?
    - 각 도메인 이벤트가 의미있는 수준으로 정의되었는가?
    - 어그리게잇: Command와 Event 들을 ACID 트랜잭션 단위의 Aggregate 로 제대로 묶었는가?
    - 기능적 요구사항과 비기능적 요구사항을 누락 없이 반영하였는가?    

  - 서브 도메인, 바운디드 컨텍스트 분리
    - 팀별 KPI 와 관심사, 상이한 배포주기 등에 따른  Sub-domain 이나 Bounded Context 를 적절히 분리하였고 그 분리 기준의 합리성이 충분히 설명되는가?
      - 적어도 3개 이상 서비스 분리
    - 폴리글랏 설계: 각 마이크로 서비스들의 구현 목표와 기능 특성에 따른 각자의 기술 Stack 과 저장소 구조를 다양하게 채택하여 설계하였는가?
    - 서비스 시나리오 중 ACID 트랜잭션이 크리티컬한 Use 케이스에 대하여 무리하게 서비스가 과다하게 조밀히 분리되지 않았는가?
  - 컨텍스트 매핑 / 이벤트 드리븐 아키텍처 
    - 업무 중요성과  도메인간 서열을 구분할 수 있는가? (Core, Supporting, General Domain)
    - Request-Response 방식과 이벤트 드리븐 방식을 구분하여 설계할 수 있는가?
    - 장애격리: 서포팅 서비스를 제거 하여도 기존 서비스에 영향이 없도록 설계하였는가?
    - 신규 서비스를 추가 하였을때 기존 서비스의 데이터베이스에 영향이 없도록 설계(열려있는 아키택처)할 수 있는가?
    - 이벤트와 폴리시를 연결하기 위한 Correlation-key 연결을 제대로 설계하였는가?

  - 헥사고날 아키텍처
    - 설계 결과에 따른 헥사고날 아키텍처 다이어그램을 제대로 그렸는가?
    
- 구현
  - [DDD] 분석단계에서의 스티커별 색상과 헥사고날 아키텍처에 따라 구현체가 매핑되게 개발되었는가?
    - Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 데이터 접근 어댑터를 개발하였는가
    - [헥사고날 아키텍처] REST Inbound adaptor 이외에 gRPC 등의 Inbound Adaptor 를 추가함에 있어서 도메인 모델의 손상을 주지 않고 새로운 프로토콜에 기존 구현체를 적응시킬 수 있는가?
    - 분석단계에서의 유비쿼터스 랭귀지 (업무현장에서 쓰는 용어) 를 사용하여 소스코드가 서술되었는가?
  - Request-Response 방식의 서비스 중심 아키텍처 구현
    - 마이크로 서비스간 Request-Response 호출에 있어 대상 서비스를 어떠한 방식으로 찾아서 호출 하였는가? (Service Discovery, REST, FeignClient)
    - 서킷브레이커를 통하여  장애를 격리시킬 수 있는가?
  - 이벤트 드리븐 아키텍처의 구현
    - 카프카를 이용하여 PubSub 으로 하나 이상의 서비스가 연동되었는가?
    - Correlation-key:  각 이벤트 건 (메시지)가 어떠한 폴리시를 처리할때 어떤 건에 연결된 처리건인지를 구별하기 위한 Correlation-key 연결을 제대로 구현 하였는가?
    - Message Consumer 마이크로서비스가 장애상황에서 수신받지 못했던 기존 이벤트들을 다시 수신받아 처리하는가?
    - Scaling-out: Message Consumer 마이크로서비스의 Replica 를 추가했을때 중복없이 이벤트를 수신할 수 있는가
    - CQRS: Materialized View 를 구현하여, 타 마이크로서비스의 데이터 원본에 접근없이(Composite 서비스나 조인SQL 등 없이) 도 내 서비스의 화면 구성과 잦은 조회가 가능한가?

  - 폴리글랏 플로그래밍
    - 각 마이크로 서비스들이 하나이상의 각자의 기술 Stack 으로 구성되었는가?
    - 각 마이크로 서비스들이 각자의 저장소 구조를 자율적으로 채택하고 각자의 저장소 유형 (RDB, NoSQL, File System 등)을 선택하여 구현하였는가?
  - API 게이트웨이
    - API GW를 통하여 마이크로 서비스들의 집입점을 통일할 수 있는가?
    - 게이트웨이와 인증서버(OAuth), JWT 토큰 인증을 통하여 마이크로서비스들을 보호할 수 있는가?
- 운영
  - SLA 준수
    - 셀프힐링: Liveness Probe 를 통하여 어떠한 서비스의 health 상태가 지속적으로 저하됨에 따라 어떠한 임계치에서 pod 가 재생되는 것을 증명할 수 있는가?
    - 서킷브레이커, 레이트리밋 등을 통한 장애격리와 성능효율을 높힐 수 있는가?
    - 오토스케일러 (HPA) 를 설정하여 확장적 운영이 가능한가?
    - 모니터링, 앨럿팅: 
  - 무정지 운영 CI/CD (10)
    - Readiness Probe 의 설정과 Rolling update을 통하여 신규 버전이 완전히 서비스를 받을 수 있는 상태일때 신규버전의 서비스로 전환됨을 siege 등으로 증명 
    - Contract Test :  자동화된 경계 테스트를 통하여 구현 오류나 API 계약위반를 미리 차단 가능한가?


# 분석/설계

## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과:  http://www.msaez.io/#/storming/vmQZPdUuSNhfyj62lkXveg5urp72/15622ad452dd80b8131472a7e471686b


### 이벤트 도출
![image](https://user-images.githubusercontent.com/20183369/135280767-12b936bc-29d4-4402-9e30-6416f1b6a7db.png)

### Actor, Command 추가
![image](https://user-images.githubusercontent.com/20183369/135279990-613edc27-0408-43cf-87b9-a7f6827eaded.png)

### Aggregate으로 묶기
![Aggregate](https://user-images.githubusercontent.com/20183369/135279509-f68b0413-9717-4e89-8b19-ec0764b84255.png)

    - 예약신청, 결제내역, 배송내역 업무영역 단위로 묶음

### Bounded Context로 묶기

![bounded_image](https://user-images.githubusercontent.com/20183369/135278745-9b8a5f78-a508-43e5-a082-81fb706a3914.png)

    - 도메인 서열 분리 
        - Core Domain:  class, course : 없어서는 안될 핵심 서비스이며, 연견 Up-time SLA 수준을 99.999% 목표, 배포주기는 class 의 경우 1주일 1회 미만, course 의 경우 1개월 1회 미만
        - General Domain:   payment : 결제서비스로 3rd Party 외부 서비스를 사용하는 것이 경쟁력이 높음 (핑크색으로 이후 전환할 예정)

### 폴리시의 이동과 컨텍스트 매핑 (점선은 Pub/Sub, 실선은 Req/Resp)

![image](https://user-images.githubusercontent.com/20183369/135278498-6f6be2cd-ffef-4833-8dbc-b4a1843e5d91.png)
    - 서비스 간의 관계 확인
      
### 완성된 1차 모형
![image](https://user-images.githubusercontent.com/20183369/135278458-e0b995e4-0eb5-43be-b4bf-10ea4acd0d79.png)

    - 서비스 간의 관계 추가

### 1차 완성본에 대한 기능적/비기능적 요구사항을 커버하는지 검증

![image](https://user-images.githubusercontent.com/49930207/131092558-df5ef712-257b-4aa4-9b49-908db9f13013.png)
    
    - 수강생이 강의를 신청한다 (ok)
    - 수강생이 강의를 결제한다 (ok)
    - 강의신청이 되면 주문 내역이 배송팀에게 전달된다 (ok)
    - 배송팀에서 강의 교재 배송 출발한다 (ok)

![image](https://user-images.githubusercontent.com/49930207/131092741-a78344ba-3b13-4ed6-a42d-b0ed005727b3.png)
    
    - 수강생이 강의를 취소할 수 있다 (ok) 
    - 강의가 취소되면 결제 취소된다 (ok) 
    - 결제가 취소되면 배송이 취소된다 (ok) 


### 모델 완료
![image](https://user-images.githubusercontent.com/20183369/135278221-bc042c91-93f1-4534-87e1-1da258cc0e9c.png)

    - 마이크로서비스별 Aggregate , Evnecnt, Policy, Command Attribute 정의
    - Naming변경
    - Mypage CQRS 정의

### 비기능 요구사항에 대한 검증

![image](https://user-images.githubusercontent.com/49930207/131438968-8c4913fa-1bf3-4461-8586-234cfa657c2f.png)

트랜잭션
강의 결제가 완료 되어야만 수강 신청 완료 할 수 있음 Sync 호출
장애격리
수강신청 시스템이 과중되면 사용자를 잠시동안 받지 않고 신청을 잠시 후에 하도록 유도한다.Circuit breaker 성능
학생은 마이페이지에서 등록된 강의와 수강 및 교재 배송 상태를 확인할 수 있어야 한다 CQRS

1. 강의 결제가 완료 되어야만 수강 신청 완료 할 수 있음 Sync 호출 및 장애격리
  - 수강생 주문시 결제처리: 결제가 완료되지 않은 주문은 절대 받지 않는다는 경영자의 오랜 신념에 따라, ACID 트랜잭션 적용. 주문완료 시 결제처리에 대해서는 Request-Response & Circuit Breaker 적용
  
2. 성능 Async. 호출 (Event Driven 방식)
  - 결제 완료시 배송처리: pay에서 course 마이크로서비스로 주문요청이 전달되는 과정에 있어서 Store 마이크로 서비스가 별도의 배포주기를 가지기 때문에 Eventual Consistency 방식으로 트랜잭션 처리함.
 - 나머지 모든 inter-microservice 트랜잭션: 주문상태, 배달상태 등 모든 이벤트에 대해 데이터 일관성의 시점이 크리티컬하지 않은 모든 경우가 대부분이라 판단, Eventual Consistency 를 기본으로 채택함.
3. 성능 Mypage (CQRS)
  - 학생용 마이페이지 및 강사용 마이페이지를 각각 구성하여 언제든 상태 정보를 확인 할 수 있음 CQRS



## 헥사고날 아키텍처 다이어그램 도출
![헥사고날 아키텍쳐 이미지](https://user-images.githubusercontent.com/88864399/133424672-e33313cf-0260-411d-a8f8-1149cb691d22.png)



  - Chris Richardson, MSA Patterns 참고하여 Inbound adaptor와 Outbound adaptor를 구분함
  - 호출관계에서 PubSub 과 Req/Resp 를 구분함
  - 서브 도메인과 바운디드 컨텍스트의 분리:  각 팀의 KPI 별로 아래와 같이 관심 구현 스토리를 나눠가짐


# 구현:

분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 Sping-Boot로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 808n 이다)

```
cd booking
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

## DDD 의 적용

- 각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다: 
 (예시는 payment 마이크로 서비스). 

```
package booking;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;
import java.util.List;
import java.util.Date;

@Entity
@Table(name="Payment_table")
public class Payment {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private Long bookingId;
    private String addr;
    private String paymentStatus;

    @PostPersist
    public void onPostPersist(){
        PaymentApproved paymentApproved = new PaymentApproved();
        BeanUtils.copyProperties(this, paymentApproved);
        paymentApproved.publishAfterCommit();

    }

    @PostUpdate
    public void onPostUpdate(){
        PayCanceled payCanceled = new PayCanceled();
        BeanUtils.copyProperties(this, payCanceled);
        payCanceled.publishAfterCommit();

    }

    @PrePersist
    public void onPrePersist(){
    }

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }
    public Long getBookingId() {
        return bookingId;
    }

    public void setBookingId(Long bookingId) {
        this.bookingId = bookingId;
    }
    public String getAddr() {
        return addr;
    }

    public void setAddr(String addr) {
        this.addr = addr;
    }
    public String getPaymentStatus() {
        return paymentStatus;
    }

    public void setPaymentStatus(String paymentStatus) {
        this.paymentStatus = paymentStatus;
    }
}

```
- Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 에 대한 별도의 처리가 없도록 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다

```
package booking;

import org.springframework.data.repository.PagingAndSortingRepository;
import org.springframework.data.rest.core.annotation.RepositoryRestResource;

import java.util.List;

@RepositoryRestResource(collectionResourceRel="payments", path="payments")
public interface PaymentRepository extends PagingAndSortingRepository<Payment, Long>{

    List<Payment> findByApplyId(String bookingId);
}

```

- 적용 후 REST API 의 테스트

```

//티켓 예약
http POST http://localhost:8081/bookings musicalName="Elizabeth" customerName="홍길동" telephoneInfo=11112222 addr="서울시 종로구"

//티켓 예약확인
http GET http://localhost:8081/bookings

//결제 확인
http GET http://localhost:8082/payments

//배송 시작 확인
http GET http://localhost:8083/deliveries

//My page 확인
http GET http://localhost:8084/mypages

//티켓 예약취소
http PATCH http://localhost:8081/bookings/1 bookingStatus=“CANCEL_BOOKING”

//티켓 예약취소 확인
http GET http://localhost:8081/bookings/1

//결제 취소 확인 (상태값 "CANCEL" 확인)
http GET http://localhost:8082/payments

//배송 취소 확인 (상태값 "DELIVERY_CANCEL" 확인)
http GET http://localhost:8083/deliveries

//My page 확인
http GET http://localhost:8084/mypages

```




## 폴리글랏 퍼시스턴스

Mypage(마이페이지) 서비스는 Maria DB 을 사용하여 구현하였다. 
Spring Cloud JPA를 사용하여 개발하였기 때문에 소스의 변경 부분은 전혀 없으며, 단지 데이터베이스 제품의 설정 (pom.xml, application.yml) 만으로 mysql 에 부착시켰다

```
# pom.yml (Delivery)

		<!-- Maria DB -->
		<dependency>
			<groupId>org.mariadb.jdbc</groupId>
			<artifactId>mariadb-java-client</artifactId>
		</dependency>


```

```
# application.yml (Mypage)

spring:
  profiles: default
  jpa:
    show_sql: true
    generate-ddl: true
    hibernate:
      ddl-auto: create-drop
  datasource:
    url: jdbc:mariadb://localhost:3306/BookingMypage
    driver-class-name: org.mariadb.jdbc.Driver
    username: root
    password: xxxx

```


- maria DB에 저장된 데이터 확인

![image](https://user-images.githubusercontent.com/20183369/135366273-2e3d8b7e-c7a7-4059-b101-a226bcf13d46.png)


## CQRS

 - 티켓 예약정보, 결제상태, 배송상태 등을 조회할 수 있도록 CQRS로 구현함
 - Booking, Payment의 Status를 통합해서 조회하기 때문에 다른 핵심 서비스들의 성능저하 이슈를 해결할 수 있다.
 - 비동기식으로 Kafka를 통해 이벤트를 수신하게 되면 별도로 관리한다

[mypage > src > main > java > team > MyPageViewHandler.java]


```
    @StreamListener(KafkaProcessor.INPUT)
    public void whenBookingStarted_then_CREATE_1 (@Payload BookingStarted bookingStarted) {
        try {

            if (!bookingStarted.validate()) return;

            // view 객체 생성
            Mypage mypage = new Mypage();
            // view 객체에 이벤트의 Value 를 set 함
            mypage.setBookingId(bookingStarted.getId());
            mypage.setMusicalName(bookingStarted.getMusicalName());
            mypage.setCustomerName(bookingStarted.getCustomerName());
            mypage.setTelephoneInfo(Long.valueOf(bookingStarted.getTelephone()));
            mypage.setAddr(bookingStarted.getAddr());
            mypage.setBookingStatus(bookingStarted.getBookingStatus());
            // view 레파지 토리에 save
            mypageRepository.save(mypage);

        }catch (Exception e){
            e.printStackTrace();
        }
    }


    @StreamListener(KafkaProcessor.INPUT)
    public void whenBookingCanceled_then_UPDATE_1(@Payload BookingCanceled bookingCanceled) {
        try {
            if (!bookingCanceled.validate()) return;
                // view 객체 조회

                    List<Mypage> mypageList = mypageRepository.findByBookingId(bookingCanceled.getId());
                    for(Mypage mypage : mypageList){
                    // view 객체에 이벤트의 eventDirectValue 를 set 함
                    mypage.setBookingStatus("CANCEL_BOOKING'");
                // view 레파지 토리에 save
                mypageRepository.save(mypage);
                }

        }catch (Exception e){
            e.printStackTrace();
        }
    }
    @StreamListener(KafkaProcessor.INPUT)
    public void whenPaymentApproved_then_UPDATE_2(@Payload PaymentApproved paymentApproved) {
        try {
            if (!paymentApproved.validate()) return;
                // view 객체 조회

                    List<Mypage> mypageList = mypageRepository.findByBookingId(paymentApproved.getBookingId());
                    for(Mypage mypage : mypageList){
                    // view 객체에 이벤트의 eventDirectValue 를 set 함
                    mypage.setPaymentStatus("COMPLETE_PAYMENT");
                // view 레파지 토리에 save
                mypageRepository.save(mypage);
                }

        }catch (Exception e){
            e.printStackTrace();
        }
    }
    @StreamListener(KafkaProcessor.INPUT)
    public void whenPayCanceled_then_UPDATE_3(@Payload PayCanceled payCanceled) {
        try {
            if (!payCanceled.validate()) return;
                // view 객체 조회

                    List<Mypage> mypageList = mypageRepository.findByBookingId(payCanceled.getBookingId());
                    for(Mypage mypage : mypageList){
                    // view 객체에 이벤트의 eventDirectValue 를 set 함
                    mypage.setPaymentStatus("CANCEL_PAYMENT");
                // view 레파지 토리에 save
                mypageRepository.save(mypage);
                }

        }catch (Exception e){
            e.printStackTrace();
        }
    }
    @StreamListener(KafkaProcessor.INPUT)
    public void whenDeliveryStarted_then_UPDATE_4(@Payload DeliveryStarted deliveryStarted) {
        try {
            if (!deliveryStarted.validate()) return;
                // view 객체 조회

                    List<Mypage> mypageList = mypageRepository.findByBookingId(deliveryStarted.getBookingId());
                    for(Mypage mypage : mypageList){
                    // view 객체에 이벤트의 eventDirectValue 를 set 함
                    mypage.setDeliveryDate(deliveryStarted.getDeliveryDate());
                    mypage.setShipperName(deliveryStarted.getShipperName());
                    mypage.setDeliveryStatus("COMPLETE_DELIVERY");
                // view 레파지 토리에 save
                mypageRepository.save(mypage);
                }

        }catch (Exception e){
            e.printStackTrace();
        }
    }
    @StreamListener(KafkaProcessor.INPUT)
    public void whenDeliveryCanceled_then_UPDATE_5(@Payload DeliveryCanceled deliveryCanceled) {
        try {
            if (!deliveryCanceled.validate()) return;
                // view 객체 조회

                    List<Mypage> mypageList = mypageRepository.findByBookingId(deliveryCanceled.getBookingId());
                    for(Mypage mypage : mypageList){
                    // view 객체에 이벤트의 eventDirectValue 를 set 함
                    mypage.setDeliveryStatus("CANCEL_DELIVERY");
                // view 레파지 토리에 save
                mypageRepository.save(mypage);
                }

        }catch (Exception e){
            e.printStackTrace();
        }
    }
```

## 동기식 호출 과 Fallback 처리

분석단계에서의 조건 중 하나로 티켓예약(booking)->결제(payment) 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다. 
호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다. 

- 결제서비스를 호출하기 위하여 Stub과 (FeignClient) 를 이용하여 Service 대행 인터페이스 (Proxy) 를 구현 


```
# (booking) PaymentService.java

package booking.external;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.stereotype.Component;

import com.netflix.hystrix.HystrixCommand;
import com.netflix.hystrix.HystrixCommandGroupKey;
import com.netflix.hystrix.HystrixCommandKey;
import com.netflix.hystrix.HystrixCommandProperties;

import feign.Feign;
import feign.hystrix.HystrixFeign;
import feign.hystrix.SetterFactory;

import java.util.Date;

@FeignClient(name="approvePay", url="http://localhost:8082", configuration=PaymentService.PaymentServiceConfiguration.class, fallback=PaymentServiceFallback.class)
public interface PaymentService {
    @RequestMapping(method= RequestMethod.POST, path="/payments")
    public boolean approvePay(@RequestBody Payment payment);

```

- booking과 payment서비스가 올라가있는 상황에서 POST 정상

![booking_post](https://user-images.githubusercontent.com/20183369/135124731-0485b8c5-26b7-4500-af99-d939d191dc38.png)
![payment_get](https://user-images.githubusercontent.com/20183369/135125122-ea888f26-a27e-44dc-ae64-b9c589b1a9a9.png)


- 동기식 호출에서는 호출 시간에 따른 타임 커플링이 발생하며, Payment가 장애가 나면 Class도 동작하지 못함을 확인

![payment서비스 장애](https://user-images.githubusercontent.com/20183369/135127745-ea97cff3-bc0e-4463-8e12-ad94fb28aa4f.png)
![classt서비스도 동작불가](https://user-images.githubusercontent.com/20183369/135127831-3f1b1bfc-cbca-4d51-887d-653a97c6163c.png)


- FallBack 처리

-1.Booking-Payment의 Request/Response 구조에 Spring Hystrix를 사용하여 FallBack 기능 구현

-2.[booking > src > main > java > Booking > external > PaymentService.java]에 configuration, fallback 옵션 추가

-3.configuration 클래스 및 fallback 클래스 추가

[booking > src > main > resources > application.yml]에 hystrix

```
# (booking) PaymentServiceFallback.java

package booking.external;

import org.springframework.stereotype.Component;

@Component
public class PaymentServiceFallback implements PaymentService {
    @Override
    public boolean approvePay(Payment payment) {

        System.out.println("\\n=========FALL BACK STARTING=========\\n"); //fallback 메소드 작동 테스트

        return false;
    }
}
```

-FallBack처리를하면, Payment장애라도 Booking 기동 중이면 정상처리됨 

![Fallback-Booking정상작동](https://user-images.githubusercontent.com/20183369/135128287-1e2f15fd-43e8-47b7-8a9c-9d745dceaef0.png)




## 비동기식 호출 / 시간적 디커플링 / 장애격리 / 최종 (Eventual) 일관성 테스트

booking(예약취소) 후 payment(결제취소) 서비스로 알려주는 행위는 동기식이 아니라 비 동기식으로 처리하여 payment(결제) 서비스 시스템 문제로 인해 예약취소 관리가 블로킹 되지 않도록 처리한다.

- 이를 위하여 예약 신청/취소 시 자체 DB에 직접 기록을 남긴 후에 곧바로 예약 신청/취소 내용을 도메인 이벤트를 카프카로 송출한다(Publish)

```
package booking;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;
import java.util.List;
import java.util.Date;

import booking.external.Payment;
import booking.external.PaymentService;

@Entity
@Table(name="Booking_table")
public class Booking {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private String musicalName;
    private String customerName;
    private Long telephoneInfo;
    private String addr;
    private String bookingStatus;

    
    @PostUpdate
    public void onPostUpdate(){
    
        System.out.println(" >> 티켓 예약취소 <<");
	
        BookingCanceled bookingCanceled = new BookingCanceled();
        BeanUtils.copyProperties(this, bookingCanceled);
        bookingCanceled.publishAfterCommit();
    }

    
```

- 예약 취소 이벤트에 대해서 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다:

```
package booking;

import booking.config.kafka.KafkaProcessor;
import com.fasterxml.jackson.databind.DeserializationFeature;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.stream.annotation.StreamListener;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.stereotype.Service;

@Service
public class PolicyHandler{
    @Autowired PaymentRepository paymentRepository;

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverBookingCanceled_CancelPay(@Payload BookingCanceled bookingCanceled){

        if(!bookingCanceled.validate()) return;

        java.util.Optional<Payment> paymentOptional = paymentRepository.findById(bookingCanceled.getId());

        Payment payment = paymentOptional.get();
        payment.setPaymentStatus("CANCEL_PAYMENT");
        paymentRepository.save(payment);


        System.out.println("\n\n##### listener CancelPay : " + bookingCanceled.toJson() + "\n\n");

    }

```
- 실제 구현을 하자면, 고객은 티켓 예약/예약취소를 한 경우, 예약상태,결제상태, 배송상태 정보를 Mypage Aggregate 내에서 조회 가능
  
```
     @StreamListener(KafkaProcessor.INPUT)
    public void whenClassCanceled_then_UPDATE_3(@Payload ClassCanceled classCanceled) {
        try {
            if (!classCanceled.validate()) return;
                // view 객체 조회
            System.out.println("\n\n##### listener classCanceled : " + classCanceled.toJson() + "\n\n");

                List<Mypage> mypageList = mypageRepository.findByApplyId(String.valueOf(classCanceled.getId()));
                for(Mypage mypage : mypageList){
                    // view 객체에 이벤트의 eventDirectValue 를 set 함
                    mypage.setApplyStaus("CancelRequest");
                // view 레파지 토리에 save
                    mypageRepository.save(mypage);
                }

        }catch (Exception e){
            e.printStackTrace();
        }
    }
    @StreamListener(KafkaProcessor.INPUT)
    public void whenPaymentCanceled_then_UPDATE_4(@Payload PaymentCanceled paymentCanceled) {
        try {
            if (!paymentCanceled.validate()) return;
                // view 객체 조회
                System.out.println("\n\n##### listener PaymentCanceled : " + paymentCanceled.toJson() + "\n\n");

                List<Mypage> mypageList = mypageRepository.findByApplyId(paymentCanceled.getApplyId());
                for(Mypage mypage : mypageList){
                    // view 객체에 이벤트의 eventDirectValue 를 set 함
                    mypage.setApplyStaus("ApplyCanceled");
                    mypage.setPayStatus("PayCanceled");
                    // view 레파지 토리에 save
                    mypageRepository.save(mypage);
                }

        }catch (Exception e){
            e.printStackTrace();
        }
    }
```



- 티켓 예약 취소 (성공) 

![비동기_예약취소](https://user-images.githubusercontent.com/20183369/135273656-e2eb72ff-ba95-417f-9128-32be2d30ba23.png)

- 결제 취소(성공)

![비동기_결제취소](https://user-images.githubusercontent.com/20183369/135273699-56b69922-4abc-40dd-9bb6-fa7398bd048e.png)

- PAYMENT(결제) 서비스가 내려가있어도 비동기식으로 BOOKING(예약 취소) 성공 되는 부분 확인
- 
![비동기_장애](https://user-images.githubusercontent.com/20183369/135273863-8c9d58a6-b3eb-46c0-b813-282deae3a64f.png)


## Gateway 

- Gateway 생성을 통하여 마이크로서비스들의 진입점을 통일시킴
[gateway > src > main > resource > application.yml]

Gateway 서비스 기동 후 각 서비스로 접근이 가능한지 확인
http GET localhost:8082/payment와 동일하게 http GET localhost:8088/payment 포트번호 바꿔서 호출 시 정상처리되면 Gatway 정상!

```
server:
  port: 8088

---

spring:
  profiles: default
  cloud:
    gateway:
      routes:
        - id: booking
          uri: http://localhost:8081
          predicates:
            - Path=/bookings/** 
        - id: payment
          uri: http://localhost:8082
          predicates:
            - Path=/payments/** 
        - id: delivery
          uri: http://localhost:8083
          predicates:
            - Path=/deliveries/** 
        - id: mypage
          uri: http://localhost:8084
          predicates:
            - Path= /mypages/**
      globalcors:
        corsConfigurations:
          '[/**]':
            allowedOrigins:
              - "*"
            allowedMethods:
              - "*"
            allowedHeaders:
              - "*"
            allowCredentials: true


```

![image](https://user-images.githubusercontent.com/20183369/135131039-fbacc06f-afd9-454f-b8b2-a278e51d238b.png)


# 운영

## CI/CD 설정

### codebuild 사용

* codebuild를 사용하여 pipeline 생성 및 배포

```  
version: 0.2

env:
  variables:
    _PROJECT_NAME: "user05-mypage"

phases:
  install:
    commands:
      - echo install kubectl
      - curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
      - chmod +x ./kubectl
      - mv ./kubectl /usr/local/bin/kubectl
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - echo $_PROJECT_NAME
      - echo $AWS_ACCOUNT_ID
      - echo $AWS_DEFAULT_REGION
      - echo $CODEBUILD_RESOLVED_SOURCE_VERSION
      - echo start command
      #- $(aws ecr get-login --no-include-email --region $AWS_DEFAULT_REGION)
      - docker login --username AWS -p $(aws ecr get-login-password --region $AWS_DEFAULT_REGION) $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/
  build:
    commands:
      - echo Build started on `date`
      - echo Building the Docker image...
      - mvn package -Dmaven.test.skip=true
      - docker build -t $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$_PROJECT_NAME:$CODEBUILD_RESOLVED_SOURCE_VERSION  .
  post_build:
    commands:
      - echo Pushing the Docker image...
      - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$_PROJECT_NAME:$CODEBUILD_RESOLVED_SOURCE_VERSION
      - echo connect kubectl
      - kubectl config set-cluster k8s --server="$KUBE_URL" --insecure-skip-tls-verify=true
      - kubectl config set-credentials admin --token="$KUBE_TOKEN"
      - kubectl config set-context default --cluster=k8s --user=admin
      - kubectl config use-context default
      - |
          cat <<EOF | kubectl apply -f -
          apiVersion: v1
          kind: Service
          metadata:
            name: mypage
            namespace: team05
            labels:
              app: mypage
          spec:
            ports:
              - port: 8080
                targetPort: 8080
            selector:
              app: mypage
          EOF
      - |
          cat  <<EOF | kubectl apply -f -
          apiVersion: apps/v1
          kind: Deployment
          metadata:
            name: mypage
            namespace: team05
            labels:
              app: mypage
          spec:
            replicas: 1
            selector:
              matchLabels:
                app: mypage
            template:
              metadata:
                labels:
                  app: mypage
              spec:
                containers:
                  - name: mypage
                    image: $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$_PROJECT_NAME:$CODEBUILD_RESOLVED_SOURCE_VERSION
                    ports:
                      - containerPort: 8080
                    readinessProbe:
                      httpGet:
                        path: '/mypages'
                        port: 8080
                      initialDelaySeconds: 20
                      timeoutSeconds: 2
                      periodSeconds: 5
                      failureThreshold: 10
                    livenessProbe:
                      httpGet:
                        path: '/mypages'
                        port: 8080
                      initialDelaySeconds: 180
                      timeoutSeconds: 2
                      periodSeconds: 5
                      failureThreshold: 5
          EOF
#cache:
#  paths:
#    - '/root/.m2/**/*'
``` 

- mypage 서비스 배포 
![image](https://user-images.githubusercontent.com/49930207/133380367-11c931d6-1fe3-43cd-bb83-fb716166fcff.png)
 
- mypage 서비스 배포 진행 단계 
![image](https://user-images.githubusercontent.com/49930207/133554065-7f2a9e2f-1d81-4aa4-a550-06073c6054c8.png)


## 동기식 호출 / 서킷 브레이킹 / 장애격리

* 서킷 브레이킹 프레임워크의 선택: Spring FeignClient + Hystrix 옵션을 사용하여 구현함

Class > Payment로 강의신청시 RESTful Request/Response 로 연동하여 구현이 되어있고, 과도할 경우 CB 를 통하여 장애격리.

- Hystrix 를 설정:  요청처리 쓰레드에서 처리시간이 1000 밀리가 넘어서기 시작하여 어느정도 유지되면 CB 회로가 닫히도록 (요청을 빠르게 실패처리, 차단) 설정
```
# application.yml

feign:
  hystrix:
    enabled: true
hystrix:
  command:
    default:
      execution.isolation.thread.timeoutInMilliseconds: 1000
```

* 부하테스터 siege 툴을 통한 서킷 브레이커 동작 확인:
- 동시사용자 50명
- 60초 동안 실시

```

root@siege:/# siege -c50 -t60S -r10 -v --content-type "application/json" 'http://localhost:8081/classes PATCH {"courseId":2}'

[error] CONFIG conflict: selected time and repetition based testing
defaulting to time-based testing: 60 seconds
** SIEGE 4.0.4
** Preparing 50 concurrent users for battle.
The server is now under siege...

HTTP/1.1 200     6.09 secs:     326 bytes ==> PATCH http://localhost:8081/classes/1
HTTP/1.1 200     5.49 secs:     326 bytes ==> PATCH http://localhost:8081/classes/1
HTTP/1.1 200     6.28 secs:     326 bytes ==> PATCH http://localhost:8081/classes/1
HTTP/1.1 200     5.40 secs:     326 bytes ==> PATCH http://localhost:8081/classes/1
HTTP/1.1 200     5.58 secs:     326 bytes ==> PATCH http://localhost:8081/classes/1


C/B 발생


HTTP/1.1 500     7.07 secs:     208 bytes ==> PATCH http://localhost:8081/classes/1
HTTP/1.1 500     3.04 secs:     208 bytes ==> PATCH http://localhost:8081/classes/1
HTTP/1.1 500     1.02 secs:     208 bytes ==> PATCH http://localhost:8081/classes/1
HTTP/1.1 500     1.03 secs:     208 bytes ==> PATCH http://localhost:8081/classes/1
HTTP/1.1 500    30.01 secs:     179 bytes ==> PATCH http://localhost:8081/classes/1

C/B 해제됨


HTTP/1.1 200    18.05 secs:     328 bytes ==> PATCH http://localhost:8081/classes/1
HTTP/1.1 200     5.10 secs:     326 bytes ==> PATCH http://localhost:8081/classes/1
HTTP/1.1 200     5.22 secs:     326 bytes ==> PATCH http://localhost:8081/classes/1
HTTP/1.1 500     4.10 secs:     239 bytes ==> PATCH http://localhost:8081/classes/1
HTTP/1.1 200     5.40 secs:     326 bytes ==> PATCH http://localhost:8081/classes/1
HTTP/1.1 200     5.70 secs:     326 bytes ==> PATCH http://localhost:8081/classes/1

...

Transactions:                   1023 hits
Availability:                  49.28 %
Elapsed time:                   8.55 secs
Data transferred:               0.56 MB
Response time:                  0.40 secs
Transaction rate:             119.65 trans/sec
Throughput:                     0.07 MB/sec
Concurrency:                   48.30
Successful transactions:        1023
Failed transactions:            1053
Longest transaction:            0.84
Shortest transaction:           0.01

```
- 운영시스템은 죽지 않고 지속적으로 CB 에 의하여 적절히 회로가 열림과 닫힘이 벌어지면서 자원을 보호하고 있음을 보여줌. 하지만, 49% 가 성공하였고, 51%가 실패했다는 것은 고객 사용성에 있어 좋지 않기 때문에 동적 Scale out (replica의 자동적 추가,HPA) 을 통하여 시스템을 확장 해주는 후속처리가 필요.

## 오토스케일 아웃
앞서 CB 는 시스템을 안정되게 운영할 수 있게 해줬지만 사용자의 요청을 100% 받아들여주지 못했기 때문에 이에 대한 보완책으로 자동화된 확장 기능을 적용하고자 한다. 


- 강좌 관리 및 강자 스케쥴 서비스에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 30프로를 넘어서면 replica 를 10개까지 늘려준다
```
kubectl autoscale deploy class --min=1 --max=10 --cpu-percent=30
```
- CB 에서 했던 방식대로 워크로드를 50초 동안 걸어준다. 
```
siege -c50 -t60S -r10 -v --content-type "application/json" 'http://localhost:8081/classes POST {"courseId":2}'
```
- 오토스케일이 어떻게 되고 있는지 모니터링을 걸어둔다:
```
watch kubectl get pod,hpa
```
- 어느정도 시간이 흐른 후 (약 30초) 스케일 아웃이 벌어지는 것을 확인할 수 있다:
```
NAME                                        REFERENCE          TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/class   Deployment/class   <unknown>/30%   1         10        3          8m53s

NAME                           READY   STATUS    RESTARTS   AGE
pod/class-5cccb78cb5-2gfdp     1/1     Running   11         50m
pod/course-7489886cf7-jpxcr    1/1     Running   0          21h
pod/delivery-c7b7d6d7d-w4crq   1/1     Running   0          148m
pod/gateway-5c8c77f4f7-wfjbh   1/1     Running   0          21h
pod/mypage-7dbb4cd488-bccw6    1/1     Running   0          50m
pod/payment-59655b4664-gfgnr   1/1     Running   0          110m
pod/siege                      1/1     Running   0          77m
:
```
- siege 의 로그를 보아도 전체적인 성공률이 높아진 것을 확인 할 수 있다. 
```
Transactions:                  27181 hits
Availability:                 100.00 %
Elapsed time:                  59.47 secs
Data transferred:               8.50 MB
Response time:                  0.11 secs
Transaction rate:             457.05 trans/sec
Throughput:                     0.14 MB/sec
Concurrency:                   49.04
Successful transactions:       27181
Failed transactions:               0
Longest transaction:            0.79
Shortest transaction:           0.00
```


## 무정지 재배포

* 먼저 무정지 재배포가 100% 되는 것인지 확인하기 위해서 Autoscaler 이나 CB 설정을 제거함

- seige 로 배포작업 직전에 워크로드를 모니터링 함.
```
siege -c200 –t120S  -v -r --content-type "application/json" 'http://localhost:8081/classes POST {"courseId":2}'

** SIEGE 4.0.5
** Preparing 100 concurrent users for battle.
The server is now under siege...

HTTP/1.1 201     3.43 secs:     251 bytes ==> POST http://localhost:8081/classes
HTTP/1.1 201     1.28 secs:     251 bytes ==> POST http://localhost:8081/classes
HTTP/1.1 201     0.20 secs:     251 bytes ==> POST http://localhost:8081/classes
HTTP/1.1 201     3.44 secs:     251 bytes ==> POST http://localhost:8081/classes
HTTP/1.1 201     1.18 secs:     251 bytes ==> POST http://localhost:8081/classes
HTTP/1.1 201     0.28 secs:     251 bytes ==> POST http://localhost:8081/classes
HTTP/1.1 201     1.41 secs:     251 bytes ==> POST http://localhost:8081/classes
HTTP/1.1 201     1.22 secs:     251 bytes ==> POST http://localhost:8081/classes
HTTP/1.1 201     0.21 secs:     251 bytes ==> POST http://localhost:8081/classes
HTTP/1.1 201     0.13 secs:     251 bytes ==> POST http://localhost:8081/classes
HTTP/1.1 201     1.41 secs:     251 bytes ==> POST http://localhost:8081/classes
HTTP/1.1 201     1.31 secs:     251 bytes ==> POST http://localhost:8081/classes

```

- 새버전(v1.0)으로의 배포 시작
```
kubectl apply -f kubectl apply -f deployment_v1.0.yml

```

- seige 의 화면으로 넘어가서 Availability 가 100% 미만으로 떨어졌는지 확인
```
Transactions:                    614 hits
Availability:                  35.35 %
Elapsed time:                  34.95 secs
Data transferred:               0.38 MB
Response time:                  3.87 secs
Transaction rate:              17.57 trans/sec
Throughput:                     0.01 MB/sec
Concurrency:                   68.06
Successful transactions:         614
Failed transactions:            1123
Longest transaction:           29.72
Shortest transaction:           0.00
```
- 배포 중 Availability 가 평소 100%에서 35% 대로 떨어지는 것을 확인. 원인은 쿠버네티스가 성급하게 새로 올려진 서비스를 READY 상태로 인식하여 서비스 유입을 진행한 것이기 때문. 이를 막기위해 Readiness Probe 를 설정함:

```
# deployment.yaml 의 readiness probe 의 설정:

# (class) deployment.yaml 파일
           readinessProbe:
            httpGet:
              path: '/classes'
              port: 8080
            initialDelaySeconds: 20
            timeoutSeconds: 2
            periodSeconds: 5
            failureThreshold: 10


/> kubectl apply -f deployment.yml
```

- 동일한 시나리오로 재배포 한 후 Availability 확인:
```
Lifting the server siege...
Transactions:                  39737 hits
Availability:                 100.00 %
Elapsed time:                 119.91 secs
Data transferred:               9.66 MB
Response time:                  0.30 secs
Transaction rate:             331.39 trans/sec
Throughput:                     0.08 MB/sec
Concurrency:                   99.71
Successful transactions:       39737
Failed transactions:               0
Longest transaction:            1.89
Shortest transaction:           0.00

```

배포기간 동안 Availability 가 변화없기 때문에 무정지 재배포가 성공한 것으로 확인됨.

## Self-healing (Liveness Probe)

- buildspec.yaml파일에 Liveness

![image](https://user-images.githubusercontent.com/88864740/133557236-8a774569-b6cd-4afb-acf2-8dba2e36129f.png)


- siege 로 Class 서비스 부하주기 (200명 10초 동시접근)

```
  siege -c200 -t10s -r5 -v --content-type "application/json" 'http://localhost:8081/classes POST {"courseId":2}'
```


![image](https://user-images.githubusercontent.com/20183369/133556725-fef2b6e0-16ce-4cd6-bfe1-5da07164c54d.png)

- 부하 후 Restart count 올라간것 확인가능

![image](https://user-images.githubusercontent.com/20183369/133556797-6f6a5dc9-e105-4a04-9aaf-50722f5438c0.png)






