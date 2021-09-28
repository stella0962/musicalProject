# 
# Musical Concert Booking
![image](https://user-images.githubusercontent.com/20183369/135119104-e58931a1-87e6-4b71-b3e1-5d4bbc38c996.png)
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
    1. 학생은 마이페이지에서 예약한 뮤지컬과 티켓 배송 상태를 확인할 수 있어야 한다  CQRS
    2. 티켓예약/배송 상태가 바뀔때마다 카톡 등으로 알림을 줄 수 있어야 한다  Event driven


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
![이벤트 도출](https://user-images.githubusercontent.com/49930207/131055626-88773fdf-77a1-4ac7-a25e-9b200fa565b1.png)

### Actor, Command 추가
![Actor, Command](https://user-images.githubusercontent.com/49930207/131058387-035d27bd-c2ff-4c08-a04a-60d361920ef6.png)

### Aggregate으로 묶기
![Aggregate](https://user-images.githubusercontent.com/49930207/131058844-53e7785f-1dd2-49e0-9bd4-1ad30af43527.png)

    - 수강신청, 강의등록, 결제내역, 배송내역 업무영역 단위로 묶음

### Bounded Context로 묶기

![Bounded Context](https://user-images.githubusercontent.com/49930207/131061428-9b1f1367-97bf-448c-b3e7-70cdcdc442af.png)

    - 도메인 서열 분리 
        - Core Domain:  class, course : 없어서는 안될 핵심 서비스이며, 연견 Up-time SLA 수준을 99.999% 목표, 배포주기는 class 의 경우 1주일 1회 미만, course 의 경우 1개월 1회 미만
        - General Domain:   payment : 결제서비스로 3rd Party 외부 서비스를 사용하는 것이 경쟁력이 높음 (핑크색으로 이후 전환할 예정)

### 폴리시의 이동과 컨텍스트 매핑 (점선은 Pub/Sub, 실선은 Req/Resp)

![Policy & Req/Resp & Pub/Sub](https://user-images.githubusercontent.com/49930207/131070356-6fbd5fbc-5b17-41a8-bcde-ca3388166f75.png)
        - 서비스 간의 관계 확인
      
### 완성된 1차 모형

![완성된 1차 모형](https://user-images.githubusercontent.com/49930207/131074402-af4a05e9-c044-4159-8b9b-26631735f05d.png)

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
![image](https://user-images.githubusercontent.com/49930207/131438677-e65d348a-775e-4121-9982-f723e1cbb1c7.png)

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
cd course
mvn spring-boot:run

cd class
mvn spring-boot:run 

cd payment
mvn spring-boot:run  

cd gateway
mvn spring-boot:run

cd delivery
mvn spring-boot:run

cd mypage
mvn spring-boot:run
```

- 아래 부터는 AWS 클라우드의 EKS 서비스 내에 서비스를 모두 배포 후 설명을 진행한다.
```
root@labs-2125792906:/home/project/team/delivery# kubectl get all -n team05
NAME                           READY   STATUS    RESTARTS   AGE
pod/class-d88578c44-jszd8      1/1     Running   0          68m
pod/course-7489886cf7-jpxcr    1/1     Running   0          68m
pod/delivery-c7b7d6d7d-svj7t   1/1     Running   0          5s
pod/gateway-5c8c77f4f7-wfjbh   1/1     Running   0          68m
pod/mypage-59bd496598-lzpxc    1/1     Running   0          54m
pod/payment-6f797bc88f-2dxl7   1/1     Running   0          68m
pod/siege                      1/1     Running   0          33m

NAME               TYPE           CLUSTER-IP       EXTERNAL-IP                                                                  PORT(S)          AGE
service/class      ClusterIP      10.100.161.69    <none>                                                                       8080/TCP         67m
service/course     ClusterIP      10.100.157.122   <none>                                                                       8080/TCP         68m
service/delivery   ClusterIP      10.100.57.150    <none>                                                                       8080/TCP         66m
service/gateway    LoadBalancer   10.100.87.132    ab6c51987a9bf4492826b76503de84b2-1875639511.ca-central-1.elb.amazonaws.com   8080:32645/TCP   57m
service/mypage     ClusterIP      10.100.101.186   <none>                                                                       8080/TCP         57m
service/payment    ClusterIP      10.100.39.160    <none>                                                                       8080/TCP         68m

NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/class      1/1     1            1           68m
deployment.apps/course     1/1     1            1           68m
deployment.apps/delivery   1/1     1            1           6s
deployment.apps/gateway    1/1     1            1           68m
deployment.apps/mypage     1/1     1            1           54m
deployment.apps/payment    1/1     1            1           68m

NAME                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/class-d88578c44      1         1         1       68m
replicaset.apps/course-7489886cf7    1         1         1       68m
replicaset.apps/delivery-c7b7d6d7d   1         1         1       6s
replicaset.apps/gateway-5c8c77f4f7   1         1         1       68m
replicaset.apps/mypage-59bd496598    1         1         1       54m
replicaset.apps/payment-6f797bc88f   1         1         1       68m
```
![클라우드EKS](https://user-images.githubusercontent.com/88864740/133535542-40d69962-f66a-4650-837e-91720ea45075.png)

## DDD 의 적용

- 각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다: 
 (예시는 payment 마이크로 서비스). 이때 가능한 중학교 수준의 영어를 사용하려고 노력했다. 

```
package team;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;

@Entity
@Table(name = "Payment_table")
public class Payment {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private Long id;
    private String applyId;
    private String payMethod;
    private String payAccount;
    private String payStaus;
    private String addr;
    private String telephoneInfo;
    private String studentName;

    @PostPersist
    public void onPostPersist() {
        PaymentAppoved paymentAppoved = new PaymentAppoved();
        BeanUtils.copyProperties(this, paymentAppoved);
        paymentAppoved.setPayStaus("PaymentAprroved");
        paymentAppoved.publishAfterCommit();
    }

    @PostUpdate
    public void onPostUpdate() {
        PaymentCanceled paymentCanceled = new PaymentCanceled();
        BeanUtils.copyProperties(this, paymentCanceled);
        paymentCanceled.publishAfterCommit();
    }

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getApplyId() {
        return applyId;
    }

    public void setApplyId(String applyId) {
        this.applyId = applyId;
    }

    public String getPayMethod() {
        return payMethod;
    }

    public void setPayMethod(String payMethod) {
        this.payMethod = payMethod;
    }

    public String getPayAccount() {
        return payAccount;
    }

    public void setPayAccount(String payAccount) {
        this.payAccount = payAccount;
    }

    public String getPayStaus() {
        return payStaus;
    }

    public void setPayStaus(String payStaus) {
        this.payStaus = payStaus;
    }

    public String getAddr() {
        return addr;
    }

    public void setAddr(String addr) {
        this.addr = addr;
    }

    public String getTelephoneInfo() {
        return telephoneInfo;
    }

    public void setTelephoneInfo(String telephoneInfo) {
        this.telephoneInfo = telephoneInfo;
    }

    public String getStudentName() {
        return studentName;
    }

    public void setStudentName(String studentName) {
        this.studentName = studentName;
    }

}

```
- Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 에 대한 별도의 처리가 없도록 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다

```
package team;

import org.springframework.data.repository.PagingAndSortingRepository;
import org.springframework.data.rest.core.annotation.RepositoryRestResource;

import java.util.List;

public interface PaymentRepository extends PagingAndSortingRepository<Payment, Long> {

    List<Payment> findByApplyId(String applyId);
}
```

- 적용 후 REST API 의 테스트

```
//강의 등록
http POST localhost:8082/courses id=1 className=English classInfo=Language teacherName=Tom teacherInfo=Male startDate=20210101 endDate=20211231

//강의 등록 확인
http GET localhost:8082/courses

//강의 삭제
http DELETE localhost:8082/courses/1

//수강 신청
http POST http://localhost:8081/classes studentName="학생2" classId="2" addr="SEOUL NAMGU" telephoneInfo="010-1234-2345" payMethod="BANK" payAccount="1234-2334-4556-7890" applyStatus="ApplyRequest"

//수강 등록 확인
http GET http://localhost:8081/classes

//결제 확인
http GET http://localhost:8083/payments

//배송 시작 확인
http GET http://localhost:8084/deliveries

//My page 확인
http GET http://localhost:8084/deliveries/mypages

//수강 취소
http PATCH http://localhost:8081/classes/1 applyStatus=“CLASS_CANCELED”

//수강 취소 확인
http GET http://localhost:8081/classes/1

//결제 취소 확인 (상태값 "CANCEL" 확인)
http GET http://localhost:8083/payments

//배송 취소 확인 (상태값 "DELIVERY_CANCEL" 확인)
http GET http://localhost:8084/deliveries

//My page 확인
http GET http://localhost:8085/mypages

```




## 폴리글랏 퍼시스턴스

Delivery(배송) 서비스는 mysql 을 사용하여 구현하였다. 
Spring Cloud JPA를 사용하여 개발하였기 때문에 소스의 변경 부분은 전혀 없으며, 단지 데이터베이스 제품의 설정 (pom.xml, application.yml) 만으로 mysql 에 부착시켰다

```
# pom.yml (Delivery)

		<dependency>
			<groupId>mysql</groupId>
			<artifactId>mysql-connector-java</artifactId>
			<version>8.0.25</version>
		</dependency>
		<dependency>

```

```
# application.yml (Delivery)

  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://mysql-1631696398.default.svc.cluster.local:3306/class?useSSL=false&characterEncoding=UTF-8&serverTimezone=UTC
    username: root
    password: RmDqZpf2rq
  jpa:
    database: mysql
    database-platform: org.hibernate.dialect.MySQL5InnoDBDialect
    generate-ddl: true
    show-sql: true 

```

- mysql 서비스 확인 (kubectl get all)

```
root@labs-2125792906:/home/project# kubectl get all
NAME                                    READY   STATUS    RESTARTS   AGE
pod/mysql-1631696398-59dbb78754-j4dt5   1/1     Running   0          85m
pod/siege-d484db9c-7x7pj                1/1     Running   0          8m16s
pod/ubuntu                              1/1     Running   0          8m21s

NAME                       TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
service/kubernetes         ClusterIP   10.100.0.1     <none>        443/TCP    91m
service/mysql-1631696398   ClusterIP   10.100.200.1   <none>        3306/TCP   85m

NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/mysql-1631696398   1/1     1            1           85m
deployment.apps/siege              1/1     1            1           8m17s

NAME                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/mysql-1631696398-59dbb78754   1         1         1       85m
replicaset.apps/siege-d484db9c                1         1         1       8m17s
```
- mysql client 에서 테스트한 데이터 확인
```
root@ubuntu:/# mysql -h mysql-1631696398 -p
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 948
Server version: 5.7.30 MySQL Community Server (GPL)

Copyright (c) 2000, 2021, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> use class
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed

mysql> show tables;
+--------------------+
| Tables_in_class    |
+--------------------+
| delivery_table     |
| hibernate_sequence |
+--------------------+
2 rows in set (0.00 sec)

mysql> select * from delivery_table
    -> ;
+----+--------+----------+-----------------+-------+------------------+
| id | addr   | apply_id | delivery_status | name  | telelephone_info |
+----+--------+----------+-----------------+-------+------------------+
|  1 | JOONGU | 1        | DeliveryStart   | name2 | 012-2345         |
+----+--------+----------+-----------------+-------+------------------+
1 row in set (0.00 sec)

```

## CQRS

 - 강의 신청정보, 결제상태, 배송상태 등을 조회할 수 있도록 CQRS로 구현함
 - Class, Payment의 Status를 통합해서 조회하기 때문에 다른 핵심 서비스들의 성능저하 이슈를 해결할 수 있다.
 - 비동기식으로 Kafka를 통해 이벤트를 수신하게 되면 별도로 관리한다

[mypage > src > main > java > team > MyPageViewHandler.java]


```
  @StreamListener(KafkaProcessor.INPUT)
    public void whenClassRegisted_then_CREATE_1 (@Payload ClassRegisted classRegisted) {
        try {

            System.out.println("\n\n##### listener classRegisted : " + classRegisted.toJson() + "\n\n");

            if (!classRegisted.validate()) return;

            // view 객체 생성
            Mypage mypage = new Mypage();
            // view 객체에 이벤트의 Value 를 set 함
            mypage.setApplyId(String.valueOf(classRegisted.getId()));
            mypage.setCourseId(String.valueOf(classRegisted.getCourseId()));
            mypage.setApplyStaus("ApplyFinish");
            mypage.setPayMethod(classRegisted.getPayMethod());
            mypage.setPayAccount(classRegisted.getPayAccount());
            mypage.setStudentName(classRegisted.getStudentName());
            mypage.setAddr(classRegisted.getAddr());
            mypage.setTelephoneInfo(classRegisted.getTelephoneInfo());
            mypage.setPayStatus("PayFinish");
            //mypage.setDeliveryStatus("D");
            // view 레파지 토리에 save
            mypageRepository.save(mypage);

        }catch (Exception e){
            e.printStackTrace();
        }
    }


    @StreamListener(KafkaProcessor.INPUT)
    public void whenPaymentAppoved_then_UPDATE_1(@Payload PaymentAppoved paymentAppoved) {
        try {
            if (!paymentAppoved.validate()) return;
                // view 객체 조회
                System.out.println("\n\n##### listener paymentAppoved : " + paymentAppoved.toJson() + "\n\n");

                List<Mypage> mypageList = mypageRepository.findByApplyId(paymentAppoved.getApplyId());
                for(Mypage mypage : mypageList){
                // view 객체에 이벤트의 eventDirectValue 를 set 함
                    mypage.setPayStatus("PayFinish");
                    mypage.setApplyStaus("ApplyFinish");
                    // view 레파지 토리에 save
                    mypageRepository.save(mypage);
                }

        }catch (Exception e){
            e.printStackTrace();
        }
    }
    @StreamListener(KafkaProcessor.INPUT)
    public void whenDeliveryStarted_then_UPDATE_2(@Payload DeliveryStarted deliveryStarted) {
        try {
            if (!deliveryStarted.validate()) return;
                // view 객체 조회
                System.out.println("\n\n##### listener deliveryStarted : " + deliveryStarted.toJson() + "\n\n");

                List<Mypage> mypageList = mypageRepository.findByApplyId(deliveryStarted.getApplyId());
                for(Mypage mypage : mypageList){
                // view 객체에 이벤트의 eventDirectValue 를 set 함

                    mypage.setDeliveryStatus("DeliveryFinish");
                // view 레파지 토리에 save
                    mypageRepository.save(mypage);
                }

        }catch (Exception e){
            e.printStackTrace();
        }
    }
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
```

## 동기식 호출 과 Fallback 처리

분석단계에서의 조건 중 하나로 강의신청(class)->결제(payment) 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다. 호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다. 

- 결제서비스를 호출하기 위하여 Stub과 (FeignClient) 를 이용하여 Service 대행 인터페이스 (Proxy) 를 구현 


```
# (class) PaymentService.java

package classnew.external;

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

@FeignClient(name="payment", url="http://localhost:8083", configuration=PaymentService.PaymentServiceConfiguration.class, fallback=PaymentServiceFallback.class)
public interface PaymentService {
    @RequestMapping(method= RequestMethod.POST, path="/payments", consumes = "application/json") //payments로 해야 데이터insert
    public boolean payApprove(@RequestBody Payment payment);
```

- class와 payment서비스가 올라가있는 상황에서 POST 정상

![class_post](https://user-images.githubusercontent.com/88864740/133542301-fc65b0f6-9328-4541-a506-828dd2514dca.png)

![payment_post](https://user-images.githubusercontent.com/88864740/133542402-9c2d2e28-65fb-47dc-8679-fe643c2219f2.png)


- 동기식 호출에서는 호출 시간에 따른 타임 커플링이 발생하며, Payment가 장애가 나면 Class도 동작하지 못함을 확인

![class-payment동기1](https://user-images.githubusercontent.com/88864740/133542529-88985589-6adc-4b49-8ba4-94bc9d3c0372.png)

![class-payment동기2](https://user-images.githubusercontent.com/88864740/133542565-354638d1-e33d-4e1f-bf38-d5c474e64579.png)


- FallBack 처리

-1.Class-Payment의 Request/Response 구조에 Spring Hystrix를 사용하여 FallBack 기능 구현

-2.[order > src > main > java > Class > external > PaymentService.java]에 configuration, fallback 옵션 추가

-3.configuration 클래스 및 fallback 클래스 추가

[class > src > main > resources > application.yml]에 hystrix

```
# (class) PaymentServiceFallback.java

package classnew.external;

import org.springframework.stereotype.Component;

@Component
public class PaymentServiceFallback implements PaymentService {
    @Override
    public boolean payApprove(Payment payment) {

        System.out.println("\\n=========FALLBACK STARTING=========\\n"); //fallback 메소드 작동 테스트
	
        return false;
    }
}
```

-FallBack처리를하면, Payment장애라도 Class기동 중이면 정상처리됨 

![image](https://user-images.githubusercontent.com/88864740/133537916-3b485d5d-10c5-4792-ad99-8e4e2bb7a4e7.png)




## 비동기식 호출 / 시간적 디커플링 / 장애격리 / 최종 (Eventual) 일관성 테스트

class(수강취소) 후 payment(결제취소) 서비스로 알려주는 행위는 동기식이 아니라 비 동기식으로 처리하여 payment(결제) 서비스 시스템 문제로 인해 수강취소 관리가 블로킹 되지 않도록 처리한다.

- 이를 위하여 수강 신청/취소 시 자체 DB에 직접 기록을 남긴 후에 곧바로 수강 신청/취소 내용을 도메인 이벤트를 카프카로 송출한다(Publish)

```
package classnew;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;
import java.util.List;
import java.util.Date;

import classnew.external.Payment;
import classnew.external.PaymentService;

@Entity
@Table(name="Class_table")
public class Class {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private String studentName;
    private String addr;
    private String telephoneInfo;
    private String payMethod;
    private String payAccount;
    private String applyStatus;
    private Long courseId;

 @PostUpdate
    public void onPostUpdate(){
        //this.setPaymentId(this.getOrderId());
        this.setApplyStatus("CLASS_CANCELED"); // path=classes의 applyStatus상태 업데이트

        ClassCanceled classCanceled = new ClassCanceled();

        classCanceled.setClassId(this.getId());
        classCanceled.setApplyStatus(this.getApplyStatus());

        BeanUtils.copyProperties(this, classCanceled);
        classCanceled.publishAfterCommit();


    }
    
```

- 강의 등록/수정 이벤트에 대해서 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다:

```
  
    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverClassCanceled_PayCancel(@Payload ClassCanceled classCanceled) {

        if (classCanceled.isMe()) {
            System.out.println("##### listener PaymentCancellation : " + classCanceled.toJson() + "\n\n");

            List<Payment> paymentList = paymentRepository.findByApplyId(String.valueOf(classCanceled.getId()));

            for (Payment payment : paymentList) {
                payment.setPayStaus("PaymentCancelled");
                paymentRepository.save(payment);
            }
        }
        return;
    }

    // Sample Logic //
    // Payment payment = new Payment();
    // paymentRepository.save(payment);

    @StreamListener(KafkaProcessor.INPUT)
    public void whatever(@Payload String eventString) {
    }

}

```
- 실제 구현을 하자면, 수강생은 수강 신청/신청취소를 한 경우, 수강신청상태,결제상태, 배송상태 정보를 Mypage Aggregate 내에서 조회 가능
  
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



- 수강 취소 (성공) 
![class_canceled](https://user-images.githubusercontent.com/88864740/133549648-2b0097c9-2149-4550-b762-76aeb59e9e26.png)

- 결제 취소(성공)
![payment_canceled](https://user-images.githubusercontent.com/88864740/133549698-2af75a62-32db-41fd-b9b1-4f182e493659.png)


- PAYMENT(결제) 서비스가 내려가있어도 비동기식으로 CLASS(수강 취소) 성공 되는 부분 확인
![비동기 payment](https://user-images.githubusercontent.com/88864740/133549843-1cd5ec1e-b0f0-4612-8a5f-5faaede3a44f.png)



## Gateway 

- Gateway 생성을 통하여 마이크로서비스들의 진입점을 통일시킴
[gateway > src > main > resource > application.yml]

Gateway 서비스 기동 후 각 서비스로 접근이 가능한지 확인
http GET localhost:8083/payment와 동일하게 http GET localhost:8088/payment 포트번호 바꿔서 호출 시 정상처리되면 Gatway 정상!

```
server:
  port: 8088

---

spring:
  profiles: default
  cloud:
    gateway:
      routes:
        - id: class
          uri: http://localhost:8081
          predicates:
            - Path=/classes/** 
        - id: course
          uri: http://localhost:8082
          predicates:
            - Path=/courses/** 
        - id: payment
          uri: http://localhost:8083
          predicates:
            - Path=/payments/** 
        - id: delivery
          uri: http://localhost:8084
          predicates:
            - Path=/deliveries/** 
        - id: mypage
          uri: http://localhost:8085
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
![image](https://user-images.githubusercontent.com/88864740/133534482-9520bba1-c659-409a-88b1-a7e6dc132b57.png)


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






