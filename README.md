# matching
선생님-학생 매칭 서비스

본 예제는 MSA/DDD/Event Storming/EDA 를 포괄하는 분석/설계/구현/운영 전단계를 커버하도록 구성한 예제입니다.
이는 클라우드 네이티브 애플리케이션의 개발에 요구되는 체크포인트들을 통과하기 위한 예시 답안을 포함합니다.
- 체크포인트 : https://workflowy.com/s/assessment-check-po/T5YrzcMewfo4J6LW



# Table of contents

- [예제 ](#---)
  - [서비스 시나리오](#서비스-시나리오)
  - [체크포인트](#체크포인트)
  - [분석/설계](#분석설계)
  - [구현:](#구현-)
    - [DDD 의 적용](#DDD-의-적용)
    - [동기식 호출 과 Fallback 처리](#동기식-호출과-Fallback-처리)
    - [이벤트드리븐 아키텍쳐의 구현](#이벤트드리븐-아키텍쳐의-구현)
    - [Poliglot](#폴리글랏-퍼시스턴스)
    - [Gateway](#Gateway)
  - [운영](#운영)
    - [CI/CD 설정](#cicd설정)
    - [동기식 호출 / 서킷 브레이킹 / 장애격리](#동기식-호출-/-서킷-브레이킹-/-장애격리)
    - [오토스케일 아웃](#오토스케일-아웃)
    - [Persistence Volume](#Persistence-Volume)
    - [Self_healing (liveness probe)](#Self_healing-(liveness-probe))
    - [무정지 재배포](#무정지-재배포)



# 서비스 시나리오

기능적 요구사항
1. 학생이 금액을 제시하여 매칭 요청을 한다. (선생님을 선택할 수 없다)
1. 학생이 결제한다.
1. 금액이 결제되면 방문요청내역이 목록에 조회된다.
1. 선생님은 방문요청을 조회한 후 선생님 이름과 만남시간을 입력하여 방문을 확정한다.
1. 학생이 매칭을 취소할 수 있다.
1. 매칭요청이 취소되면 방문이 취소된다.
1. 학생은 myPage에서 매칭 상태를 중간중간 조회할 수 있다.
1. 매칭요청 화면에서 상태를 조회할 수 있다. 
1. 매칭요청/결제요청/방문확정/결제취소/방문취소 시 상태가 변경된다.
1. 선생님이 방문 완료 요청한다. 
1. 수업 상태 화면에서 선생님의 수업 완료 상태를 확인할 수 있다.

비기능적 요구사항
1. 트랜잭션
    1. 결제가 되지 않은 매칭건은 아예 매칭이 성립되지 않아야 한다. Sync 호출
1. 장애격리
    1. 방문관리 기능이 수행되지 않더라도 매칭요청은 365/24 받을 수 있어야 한다. Async(event-driven) Eventual Consistency
1. 성능
    1. 학생이 매칭시스템에서 확인할 수 있는 상태를 마이페이지(프론트엔드)에서 확인할 수 있어야 한다 CQRS
    1. 상태가 바뀔때마다 myPage에서는 변경된 상태를 조회할 수 있어야 한다. Event driven
    1. 방문 상태가 변경될 때 마다 매칭관리에서 상태가 변경되어야 한다. corelation
    1. 방문 완료되면 수업 상태 화면에서 선생님의 수업 완료 상태를 확인할 수 있어야 한다. corelation



# 분석/설계

## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과:  http://www.msaez.io/#/storming/xEZmSDJKirOi8JxZbHu3ZOJMmQY2/every/701ca815b3e6ac4e15668ef609e86f43


## 이벤트 도출/Saga

### 최종 이벤트스토밍 결과
![image](https://user-images.githubusercontent.com/75401933/105022842-8e58b980-5a8d-11eb-868c-aae24f8db3ed.png)

### 개인 이벤트스토밍 결과
![이벤트스토밍_je](https://user-images.githubusercontent.com/45473909/105210254-2c33ad80-5b8e-11eb-9349-5fcdcba17ded.PNG)


## 헥사고날 아키텍처 다이어그램 도출
![헥사고날아키텍처_je](https://user-images.githubusercontent.com/45473909/105210298-3786d900-5b8e-11eb-8937-5c0be0090db5.PNG)



# 구현:

분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 8085 이다)

```
cd match
mvn spring-boot:run

cd visit
mvn spring-boot:run  

cd payment
mvn spring-boot:run 

cd mypage
mvn spring-boot:run  

cd class
mvn spring-boot:run  
```


## DDD 의 적용

- 각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다: (예시는 match 마이크로 서비스). 이때 가능한 현업에서 사용하는 언어 (유비쿼터스 랭귀지)를 그대로 사용하였고, 모든 구현에 있어서 영문으로 사용하여 별다른  오류없이 구현하였다.

```
package matching;

import javax.persistence.*;

import matching.external.Payment;
import matching.external.PaymentService;
import org.springframework.beans.BeanUtils;
import java.util.List;

@Entity
@Table(name="Match_table")
public class Match {

    @Id
    private Long id;
    private Integer price;
    private String status;

    @PostPersist
    public void onPostPersist(){
        MatchRequested matchRequested = new MatchRequested();
        BeanUtils.copyProperties(this, matchRequested);
        matchRequested.publishAfterCommit();

        //Following code causes dependency to external APIs
        // it is NOT A GOOD PRACTICE. instead, Event-Policy mapping is recommended.
        Payment payment = new Payment();
        // mappings goes here

        //변수 setting
        payment.setMatchId(Long.valueOf(this.getId()));
        payment.setPrice(Integer.valueOf(this.getPrice()));
        payment.setPaymentAction("Approved");

        MatchApplication.applicationContext.getBean(PaymentService.class)
                .paymentRequest(payment);
    }

    @PreUpdate
    public void onPreUpdate(){
        if("cancel".equals(status)) {
            MatchCanceled matchCanceled = new MatchCanceled();
            BeanUtils.copyProperties(this, matchCanceled);
            matchCanceled.publishAfterCommit();
        }
    }

    public Long getId() {
        return id;
    }
    public void setId(Long id) {
        this.id = id;
    }

    public Integer getPrice() {
        return price;
    }
    public void setPrice(Integer price) {
        this.price = price;
    }

    public String getStatus() { return status; }
    public void setStatus(String status) {
        this.status = status;
    }
}

```
- Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 에 대한 별도의 처리가 없도록 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다
```
package matching;

import org.springframework.data.repository.PagingAndSortingRepository;

public interface MatchRepository extends PagingAndSortingRepository<Match, Long>{
}
```

- 적용 후 REST API 의 테스트
```
# match 서비스의 접수처리
http localhost:8088/matches id=5000 price=20000 status=matchRequest
```
![1 매치날림](https://user-images.githubusercontent.com/45473909/105169628-2377b300-5b5f-11eb-8b31-88c88b23be0c.PNG)
```
# visit에서 선생님의 방문 일정 확정
http POST localhost:8088/visits matchId=5000 teacher=kim visitDate=210101
```
![2 비지트날림](https://user-images.githubusercontent.com/45473909/105169631-24a8e000-5b5f-11eb-861a-26a85e956ef2.PNG)
```
# payment 서비스의 상태확인
http localhost:8088/payments/5000
```
![3 페이먼트확인](https://user-images.githubusercontent.com/45473909/105169640-25da0d00-5b5f-11eb-95b8-bd00fe9595a9.PNG)
```
# myPage에서 match 서비스, visit 서비스 상태 확인
http localhost:8088/myPages/5000
```
![4 마이페이지확인](https://user-images.githubusercontent.com/45473909/105169645-27a3d080-5b5f-11eb-8487-48a314b14174.PNG)
```
# visit에서 승인된 일정 classStatus에서 확인
http localhost:8088/classStatuses/5000
```
![5 클래스에선생님정보](https://user-images.githubusercontent.com/45473909/105169648-296d9400-5b5f-11eb-9850-e535c1153145.PNG)
```
# classStatus에서 방문 완료 접수처리
http localhost:8088/classStatuses id=5000 status=classFinished
```
![6 클래스에수업상태](https://user-images.githubusercontent.com/45473909/105169653-2b375780-5b5f-11eb-9676-cca87ae082df.PNG)


### CQRS

수업 상태가 변하면 class 서비스 및 classStatus view에서 확인할 수 있도록 구현하였다.  

```
# class > ClassStatusViewHandler.java

 @StreamListener(KafkaProcessor.INPUT)
    public void wheneverVisitAssigned_(@Payload final VisitAssigned visitAssigned){

        if(visitAssigned.isMe()){
            System.out.println("##### listener  : " + visitAssigned.toJson());

            final ClassStatus classStatus = new ClassStatus();

            classStatus.setId(visitAssigned.getMatchId());
            classStatus.setTeacher(visitAssigned.getTeacher());
            classStatus.setVisitDate(visitAssigned.getVisitDate());
            classStatusRepository.save(classStatus);
        }
    }
    

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverClassFinished_(@Payload final ClassFinished classFinished){

        if(classFinished.isMe()){
            System.out.println("##### listener  : " + classFinished.toJson());

            classStatusRepository.findById(classFinished.getId()).ifPresent(cclassStatus ->{
                cclassStatus.setStatus(classFinished.getClassStatus());
                classStatusRepository.save(cclassStatus);

            });
        }
    }
}
    
```
- class의 view로 확인
![class 첫화면](https://user-images.githubusercontent.com/45473909/105314095-68ecbc80-5c01-11eb-9cb9-cce0d09956bf.PNG)

![class 화면](https://user-images.githubusercontent.com/45473909/105314121-6ab68000-5c01-11eb-93b2-6890984d28a7.PNG)

![classS 첫화면](https://user-images.githubusercontent.com/45473909/105314147-6be7ad00-5c01-11eb-8557-626fa5e67485.PNG)

![classS에서 들어온거](https://user-images.githubusercontent.com/45473909/105314161-6c804380-5c01-11eb-8c3d-2eee6185c9cf.PNG)

![class로 상태 날리고](https://user-images.githubusercontent.com/45473909/105314187-6e4a0700-5c01-11eb-986f-26743c5b0949.PNG)

![class에서 들어온거](https://user-images.githubusercontent.com/45473909/105314208-6f7b3400-5c01-11eb-8838-7d6a728c833a.PNG)

### SAGA / Corelation

방문(visit) 시스템에서 상태가 방문확정 또는 방문취소로 변경되면 매치(match) 시스템 원천데이터의 상태(status) 정보가 update된다.  

```
# mypage > PolicyHandler.java

  @StreamListener(KafkaProcessor.INPUT)
  public void wheneverVisitAssigned_(@Payload VisitAssigned visitAssigned){

      if(visitAssigned.isMe()){
          System.out.println("##### listener wheneverVisitAssigned  : " + visitAssigned.toJson());

          //방문 assign 이벤트를 수신하였을 때 해당 ID를 찾아서 상태값을 visitAssigned로 변경
          MatchRepository.findById(visitAssigned.getMatchId()).ifPresent(Match ->{
              System.out.println("##### wheneverVisitAssigned_MatchRepository.findById : exist" );

              Match.setStatus(visitAssigned.getEventType()); //상태값은 모두 이벤트타입으로 셋팅함
              MatchRepository.save(Match);
          });

      }
  }


  @StreamListener(KafkaProcessor.INPUT)
  public void wheneverVisitCanceled_(@Payload VisitCanceled visitCanceled) {

      if (visitCanceled.isMe()) {
          System.out.println("##### listener  : " + visitCanceled.toJson());

          //방문취소 이벤트를 수신하였을 때 해당 ID를 찾아서 상태값을 visitCanceled로 변경
          MatchRepository.findById(visitCanceled.getMatchId()).ifPresent(Match -> {
              System.out.println("##### wheneverVisitCanceled_MatchRepository.findById : exist");
              Match.setStatus(visitCanceled.getEventType());
              MatchRepository.save(Match);
          });
      }
  }

```

![1 매치날림](https://user-images.githubusercontent.com/45473909/105169628-2377b300-5b5f-11eb-8b31-88c88b23be0c.PNG)
![match테스트](https://user-images.githubusercontent.com/45473909/105259293-e2b78280-5bce-11eb-8c2d-051f8a9bace0.PNG)

## 동기식 호출과 Fallback 처리

분석단계에서의 조건 중 하나로 접수(match)->결제(payment) 간의 호출은 동기식으로 호출하고자  동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다. 호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient를 이용하여 호출하도록 한다.

- 결제서비스를 호출하기 위하여 FeignClient 를 이용하여 Service 대행 인터페이스 (Proxy) 를 구현

```
# (match) PaymentService.java

package matching.external;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;

import java.util.Date;

@FeignClient(name="payment", url="${api.payment.url}")
public interface PaymentService {

    @RequestMapping(method= RequestMethod.POST, path="/payments")
    public void paymentRequest(@RequestBody Payment payment);

}
```

- match접수를 받은 직후(@PostPersist) 결제를 요청하도록 처리
```
# match.java (Entity)
  @PostPersist
    public void onPostPersist(){
        MatchRequested matchRequested = new MatchRequested();
        BeanUtils.copyProperties(this, matchRequested);
        matchRequested.publishAfterCommit();

        //Following code causes dependency to external APIs
        // it is NOT A GOOD PRACTICE. instead, Event-Policy mapping is recommended.
        Payment payment = new Payment();
        // mappings goes here

        //변수 setting
        payment.setMatchId(Long.valueOf(this.getId()));
        payment.setPrice(Integer.valueOf(this.getPrice()));
        payment.setPaymentAction("Approved");

        MatchApplication.applicationContext.getBean(PaymentService.class)
                .paymentRequest(payment);
    }
```

- 동기식 호출에서는 호출 시간에 따른 타임 커플링이 발생하며, 결제 시스템이 장애가 나면 접수도 못받는다는 것을 확인


```
# 결제 (payment) 서비스를 잠시 내려놓음 (ctrl+c)

# 접수처리
http localhost:8088/matches id=5005 price=50000 status=matchRequest   #Fail
```
![11 payment내리면match안됨](https://user-images.githubusercontent.com/45473909/105013488-a7a83880-5a82-11eb-9417-92d92668b879.PNG)
```

# payment서비스 재기동
cd payment
mvn spring-boot:run

#match 처리
http localhost:8088/matches id=5006 price=50000 status=matchRequest  #Success
```
![11 payment올리면match됨](https://user-images.githubusercontent.com/45473909/105013494-a8d96580-5a82-11eb-95de-73a47f072920.PNG)


## 이벤트드리븐 아키텍쳐의 구현

### 비동기식 호출 

결제가 완료 된 후에 방문(visit) 시스템으로 이를 알려주는 행위는 동기식이 아닌 비동기식으로 처리하며, 방문시스템의 처리를 위하여 매칭요청/결제가 블로킹 되지 않도록 처리한다.
 
- 이를 위하여 결제요청이력에 기록을 남긴 후에 곧바로 결제 완료 되었다는 도메인 이벤트를 카프카로 송출한다(Publish)
 
```
package matching;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;
import java.util.List;

@Entity
@Table(name="Payment_table")
public class Payment {

    @Id
    private Long matchId;
    private Integer price;
    private String paymentAction;

    @PostPersist
    public void onPostPersist(){
        PaymentApproved paymentApproved = new PaymentApproved();
        BeanUtils.copyProperties(this, paymentApproved);
        paymentApproved.publishAfterCommit();
    }


```

- 방문 서비스에서는 결제완료 이벤트를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다:

```
package matching;

import matching.config.kafka.KafkaProcessor;
import com.fasterxml.jackson.databind.DeserializationFeature;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.stream.annotation.StreamListener;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.stereotype.Service;

@Service
public class PolicyHandler{
    @Autowired VisitReqListRepository VisitReqListRepository;
    @Autowired VisitRepository VisitRepository;

    @StreamListener(KafkaProcessor.INPUT)
    public void onStringEventListener(@Payload String eventString){

    }

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverPaymentApproved_(@Payload PaymentApproved paymentApproved){

        if(paymentApproved.isMe()){
            System.out.println("##### listener  : " + paymentApproved.toJson());


            //승인완료 시 승인완료된 리스트를 visitReqList에 받아서 보여줄 수 있도록 id setting

            VisitReqList visitReqList = new VisitReqList();

            visitReqList.setId(paymentApproved.getMatchId());
            VisitReqListRepository.save(visitReqList);

        }
    }

```

### 시간적 디커플링 / 장애격리 

방문(visit) 시스템은 결제(payment) 시스템과 완전히 분리되어있으며 이벤트 수신에 따라 처리되기 때문에, 방문 시스템이 유지보수로 인해 잠시 내려간 상태라도 방문요청(match) 및 결제(payment)하는데에 문제가 없다


- 방문 서비스(visit)를 잠시 놓은 후 매칭 요청 처리
```
# 매칭요청 처리
http POST http://localhost:8081/matches id=101 price=5000 status=matchRequest   #Success
```
![비지트 내리고  매치](https://user-images.githubusercontent.com/45473909/105259784-c8ca6f80-5bcf-11eb-89f2-b1c147dc78e5.PNG)

- 결제서비스가 정상적으로 조회되었는지 확인
```
http http://localhost:8083/payments   #Success
```
![비지트 내리고 페이먼트](https://user-images.githubusercontent.com/45473909/105259781-c700ac00-5bcf-11eb-8951-7869a55166f9.PNG)

- 방문 서비스 다시 가동
```
cd visit
mvn spring-boot:run
```

- 가동 전/후의 방문상태 확인
```
# 신규 접수된 매칭요청건에 대해 선생님과 방문일자 매칭
http POST http://localhost:8082/visits matchId=20 teacher=kim visitDate=20210101 
http localhost:8082/visits     
```
![비지트다시올리고포스트](https://user-images.githubusercontent.com/45473909/105259782-c831d900-5bcf-11eb-9ffe-ace968715e7b.PNG)
![비지트다시올리고확인](https://user-images.githubusercontent.com/45473909/105259783-c831d900-5bcf-11eb-8367-1a8564fa95f1.PNG)


## Gateway

```
server:
  port: 8088

---

spring:
  profiles: default
  cloud:
    gateway:
      routes:
        - id: match
          uri: http://localhost:8081
          predicates:
            - Path=/matches/** 
        - id: visit
          uri: http://localhost:8082
          predicates:
            - Path=/visits/**, /visitReqLists/**
        - id: payment
          uri: http://localhost:8083
          predicates:
            - Path=/payments/** 
        - id: mypage
          uri: http://localhost:8084
          predicates:
            - Path=/myPages/**, /myPages/**
        - id: class
          uri: http://localhost:8085
          predicates:
            - Path=/classes/**, /classStatuses/**
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

---

spring:
  profiles: docker
  cloud:
    gateway:
      routes:
        - id: match
          uri: http://match:8080
          predicates:
            - Path=/matches/** 
        - id: visit
          uri: http://visit:8080
          predicates:
            - Path=/visits/**, /visitReqLists/**
        - id: payment
          uri: http://payment:8080
          predicates:
            - Path=/payments/** 
        - id: mypage
          uri: http://mypage:8080
          predicates:
            - Path=/myPages/**, /myPages/**
        - id: class
          uri: http://class:8080
          predicates:
            - Path=/classes/**, /classStatuses/**
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

server:
  port: 8080
```

- Gateway 서비스 실행 상태에서 8088과 8081로 각각 서비스 실행하였을 때 동일하게 match 서비스 실행되었다.
```
http localhost:8088/matches id=50 price=50000 status=matchRequest
```
![8088포트](https://user-images.githubusercontent.com/45473909/105039570-0f22b000-5aa4-11eb-9090-45662dcd79d0.PNG)

```
http localhost:8081/matches id=51 price=50000 status=matchRequest
```
![8081포트](https://user-images.githubusercontent.com/45473909/105039551-0a5dfc00-5aa4-11eb-86c0-c3fc63d5b0f6.PNG)


## 폴리글랏 퍼시스턴스

match 는 다른 서비스와 구별을 위해 별도 hsqldb를 사용 하였다. 이를 위해 match내 pom.xml에 dependency를 h2database에서 hsqldb로 변경 하였다.

```
#match의 pom.xml dependency를 수정하여 DB변경

  <!--
  <dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>runtime</scope>
  </dependency>
  -->

  <dependency>
    <groupId>org.hsqldb</groupId>
    <artifactId>hsqldb</artifactId>
    <version>2.4.0</version>
    <scope>runtime</scope>
  </dependency>

```



# 운영

## CI/CD 설정
각 구현체들은 각자의 source repository 에 구성되었고, 사용한 CI/CD 플랫폼은 Azure를 사용하였으며, pipeline build script 는 각 프로젝트 폴더 이하에 deployment.yml, service.yml 에 포함되었다

![CI](https://user-images.githubusercontent.com/45473909/105212831-4c18a080-5b91-11eb-8656-1a2c6d5225ee.PNG)

![CD](https://user-images.githubusercontent.com/45473909/105212838-4e7afa80-5b91-11eb-85d2-1a43849fd84f.PNG)

![레파지토리](https://user-images.githubusercontent.com/45473909/105269927-35496c80-5bd8-11eb-8334-138149a79a01.PNG)
![repo](https://user-images.githubusercontent.com/45473909/105270011-5f9b2a00-5bd8-11eb-9c31-d2d0bc79bfbd.PNG)


## 오토스케일 아웃

visit 구현체에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 10프로를 넘어서면 replica 를 10개까지 늘려준다:

kubectl autoscale deploy visit --min=1 --max=10 --cpu-percent=10

kubectl exec -it pod siege -- /bin/bash

siege -c20 -t120S -v http://visit:8080/visits/600

부하에 따라 visit pod의 cpu 사용률이 증가했고, Pod Replica 수가 증가하는 것을 확인할 수 있었다.
![오토스ㅔ일](https://user-images.githubusercontent.com/45473909/105271240-7fcbe880-5bda-11eb-83ad-7736a3d7234f.PNG)


