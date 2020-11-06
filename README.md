![image](https://user-images.githubusercontent.com/70673885/97950284-bcf1bd00-1dd9-11eb-8c8a-b3459c710849.png)
ver.Answan

# 서비스 시나리오

기능적 요구사항
1. 고객이 APP에서 폰을 주문한다.
1. 고객이 결제한다.
1. 결제가 완료되면 Reward를 지급한다.
1. 주문이 되면 주문 내역이 대리점에 전달된다.
1. 대리점에 주문 정보가 도착하면 배송한다.
1. 배송이 되면 APP에서 배송상태를 조회할 수 있다.
1. 고객이 주문을 취소할 수 있다.
1. 주문이 취소되면 결제가 취소된다.
2. 결재가 취소되면 Reward를 차감한다.
1. 고객이 결제상태를 APP에서  조회 할 수 있다.
1. 고객이 모든 진행내역을 볼 수 있어야 한다.

비기능적 요구사항
1. 트랜잭션    
    1. 결제가 취소되면 Reward는 무조건 회수한다.> Sync 호출(개인)
    1. 결제가 처리되면 Reward가 전송되고 주문정보에 업데이트가 되어야 한다.> SAGA, 보상 트랜젝션
    
1. 장애격리
    1. Reward관리 기능이 수행되지 않더라도 주문은 365일 24시간 받을 수 있어야 한다.> Async (event-driven), Eventual Consistency
       (Sync인 주문 취소는 수행되지 않는다.)
    1. Reward관리시스템이 과중되면 주문을 잠시동안 받지 않고 결제를 잠시후에 하도록 유도한다> Circuit breaker, fallback
1. 성능
    1. 고객이 모든 진행내역과 포인트 변동 내역을 조회 할 수 있도록 성능을 고려하여 별도의 view로 구성한다.> CQRS


# 체크포인트

1. Saga
1. CQRS
1. Correlation
1. Req/Resp
1. Gateway
1. Deploy/ Pipeline
1. Circuit Breaker
1. Autoscale (HPA)
1. Zero-downtime deploy (Readiness Probe)
1. Config Map/ Persistence Volume
1. Polyglot
1. Self-healing (Liveness Probe)


# 분석/설계


## AS-IS 조직 (Horizontally-Aligned)
  ![image](https://user-images.githubusercontent.com/487999/79684144-2a893200-826a-11ea-9a01-79927d3a0107.png)

## TO-BE 조직 (Vertically-Aligned)
  ![image](https://user-images.githubusercontent.com/487999/79684159-3543c700-826a-11ea-8d5f-a3fc0c4cad87.png)


## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과:  http://www.msaez.io/#/storming/dukpFsKpNqU89oVjZp4DTcUnsXM2/share/f684312518e30472bbeaf470fa8fe6d3/-MLLAnpSdww_03eFjFnJ


### 이벤트 도출
![image](https://user-images.githubusercontent.com/52647474/98241971-0d6e4380-1faf-11eb-9615-4dfd74f6581a.png)

### 부적격 이벤트 탈락
![image](https://user-images.githubusercontent.com/52647474/98242134-54f4cf80-1faf-11eb-8cca-d920cf733e8b.png)

    - 과정중 도출된 잘못된 도메인 이벤트들을 걸러내는 작업을 수행함
	- 폰종류가선택됨, 결제버튼클릭됨, 배송수량선택됨, 배송일자선택됨,Reward 승인클릭 :  UI 의 이벤트이지, 업무적인 의미의 이벤트가 아니라서 제외
	- 배송취소됨, 메시지발송됨,취소메시지 발송  :  계획된 사업 범위 및 프로젝트에서 벗어서난다고 판단하여 제외
	- 주문정보전달됨  :  주문됨을 선택하여 제외
	

### 액터, 커맨드 부착하여 읽기 좋게
![image](https://user-images.githubusercontent.com/52647474/98242215-7655bb80-1faf-11eb-9385-383b7ed2e7b4.png)

### 어그리게잇으로 묶기
![image](https://user-images.githubusercontent.com/52647474/98242269-89688b80-1faf-11eb-89c7-c7370db3b7ca.png)

    - 주문, 대리점관리, 결제, Reward 어그리게잇을 생성하고 그와 연결된 command 와 event 들에 의하여 트랜잭션이 유지되어야 하는 단위로 그들 끼리 묶어줌

### 바운디드 컨텍스트로 묶기

![image](https://user-images.githubusercontent.com/52647474/98242633-190e3a00-1fb0-11eb-9a94-18caa6101e15.png)

    - 도메인 서열 분리 
        - Core Domain:  app(front), store : 없어서는 안될 핵심 서비스이며, 연견 Up-time SLA 수준을 99.999% 목표, 배포주기는 app 의 경우 1주일 1회 미만, store 의 경우 1개월 1회 미만
        - Supporting Domain:  customer(view) : 경쟁력을 내기위한 서비스이며, SLA 수준은 연간 60% 이상 uptime 목표, 배포주기는 각 팀의 자율이나 표준 스프린트 주기가 1주일 이므로 1주일 1회 이상을 기준으로 함.
        - General Domain: Reward, pay : 결제서비스와 Reward서비스로 3rd Party 외부 서비스를 사용하는 것이 경쟁력이 높음 

### 폴리시 부착 (괄호는 수행주체, 폴리시 부착을 둘째단계에서 해놔도 상관 없음. 전체 연계가 초기에 드러남)

![image](https://user-images.githubusercontent.com/52647474/98242443-d8162580-1faf-11eb-85a5-12d0db71ab51.png)

### 폴리시의 이동

![image](https://user-images.githubusercontent.com/52647474/98244875-72c43380-1fb3-11eb-8b68-09b052c7f1df.png)

### 컨텍스트 매핑 (점선은 Pub/Sub, 실선은 Req/Resp)

![image](https://user-images.githubusercontent.com/52647474/98245367-288f8200-1fb4-11eb-8f12-239f7c7a86f2.png)

    - 컨텍스트 매핑하여 묶어줌.
    - 팀원 중 외국인이 투입되어 유비쿼터스 랭귀지인 영어로 변경	

### 완성된 모형

![image](https://user-images.githubusercontent.com/52647474/98245482-54ab0300-1fb4-11eb-8bc6-bc5bdb403388.png)

    - View Model 포함

### 기능적 요구사항 검증

![image](https://user-images.githubusercontent.com/52647474/98245982-09452480-1fb5-11eb-9aa9-b5bf29ca641c.png)

   	- 고객이 APP에서 폰을 주문한다. (ok)
   	- 고객이 결제한다. (ok)
	- 결제가 되면 결제완료 내역이 Reward에 전달된다. (ok)
	- Reward에 결제완료 정보가 도착하면 Reward를 지급 한다. (ok)
	- Reward지급이 되면 APP에서 Reward를 조회할 수 있다. (ok)

![image](https://user-images.githubusercontent.com/52647474/98246085-25e15c80-1fb5-11eb-861e-ed0255bdbf28.png)

	- 고객이 주문을 취소할 수 있다. (ok)
	- 결재가 취소되면 Reward가 취소된다. (ok)
	- 고객이 Reward를 APP에서  조회 할 수 있다. (ok)

![image](https://user-images.githubusercontent.com/52647474/98246348-896b8a00-1fb5-11eb-8cce-4a9476a6ba73.png)
  
	- 고객이 모든 진행내역을 볼 수 있어야 한다. (ok)


### 비기능 요구사항 검증

![image](https://user-images.githubusercontent.com/52647474/98246898-44942300-1fb6-11eb-8ca4-bf99838feb6e.png)

    - 1) 결제가 취소되지 않은건은 Reward를 회수할수 없어야한다. (Req/Res)
    - 2) Reward관리 기능이 수행되지 않더라도 주문은 365일 24시간 받을 수 있어야 한다. (Pub/sub)
    - 3) Reward시스템이 과중되면 사용자를 잠시동안 받지 않고 결제를 잠시후에 하도록 유도한다. (Circuit breaker)
    - 4) 결제가 완료되면 Rewar가 지급되고 주문정보에 업데이트가 되어야 한다.  (SAGA, 보상트렌젝션)
    - 5) 고객이 모든 진행내역을 조회 할 수 있도록 성능을 고려하여 별도의 view로 구성한다. (CQRS, DML/SELECT 분리)결제가 취소되면 Reward는 무조건 회수한다.> Sync 호출(개인)



## 헥사고날 아키텍처 다이어그램 도출 (Polyglot)

![image](https://user-images.githubusercontent.com/52647474/98247802-6fcb4200-1fb7-11eb-87dc-9e5e7e32707d.png)

    - Chris Richardson, MSA Patterns 참고하여 Inbound adaptor와 Outbound adaptor를 구분함
    - 호출관계에서 PubSub 과 Req/Resp 를 구분함
    - 서브 도메인과 바운디드 컨텍스트의 분리:  각 팀의 KPI 별로 아래와 같이 관심 구현 스토리를 나눠가짐
    - 대리점의 경우 Polyglot 검증을 위해 Hsql로 셜계


# 구현:

서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 808n 이다)

```
cd reward
mvn spring-boot:run

cd app
mvn spring-boot:run

cd pay
mvn spring-boot:run 

cd store
mvn spring-boot:run  

cd customer
mvn spring-boot:run  
```

## DDD 의 적용

각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다: (예시는 Reward 마이크로 서비스). 
이때 가능한 현업에서 사용하는 언어 (유비쿼터스 랭귀지)를 그대로 사용하려고 노력했다. 
하지만, 일부 구현에 있어서 영문이 아닌 경우는 실행이 불가능한 경우가 있기 때문에 계속 사용할 방법은 아닌것 같다. 
(Maven pom.xml, Kafka의 topic id, FeignClient 의 서비스 id 등은 한글로 식별자를 사용하는 경우 오류가 발생하는 것을 확인하였다)

![image](https://user-images.githubusercontent.com/52647474/98247962-a903b200-1fb7-11eb-9ce7-85dbbae4bafd.png)

Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 에 대한 별도의 처리가 없도록 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다

![image](https://user-images.githubusercontent.com/52647474/98248187-f122d480-1fb7-11eb-9c49-82560ccffe3a.png)


## 폴리글랏 퍼시스턴스
대리점의 경우 H2 DB인 주문과 결제와 달리 Hsql으로 구현하여 MSA간 서로 다른 종류의 DB간에도 문제 없이 동작하여 다형성을 만족하는지 확인하였다. 


app, pay,store, customer의 pom.xml 설정

![image](https://user-images.githubusercontent.com/73699193/97972993-baf32280-1e08-11eb-8158-912e4d28d7ea.png)


reward의 pom.xml 설정

![image](https://user-images.githubusercontent.com/52647474/98248333-22030980-1fb8-11eb-9ab3-0edeb77ee4ef.png)



## Gateway 적용

gateway > applitcation.yml 설정

![image](https://user-images.githubusercontent.com/52647474/98248509-6098c400-1fb8-11eb-9f8d-df5492855827.png)

gateway 테스트

```
http POST http://gateway:8080/orders item=test qty=1
```
![image](https://user-images.githubusercontent.com/52647474/98259835-eec77700-1fc5-11eb-9ac9-bab6a80a0bb0.png)

![image](https://user-images.githubusercontent.com/52647474/98259741-d6575c80-1fc5-11eb-8335-e048a8a01333.png)



## 동기식 호출 과 Fallback 처리

분석단계에서의 조건 중 하나로 결제(pay)->Reward 간의 결제취소 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다. 
호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다. 

- Reward서비스를 호출하기 위하여 FeignClient 를 이용하여 Service 대행 인터페이스 (Proxy) 를 구현 
```
# (reward) external > RewardService.java


package phoneseller.external;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;

import java.util.Date;

@FeignClient(name="reward", url="${api.url.reward}")
public interface RewardService {

    @RequestMapping(method= RequestMethod.POST, path="/rewards")
    public void payCancel(@RequestBody Reward reward);

}
```
![image](https://user-images.githubusercontent.com/73699193/98065833-b1190000-1e98-11eb-9e44-84d4961011ed.png)


- 결제취소를 받은 직후 Reward를 포인트를 회수 하도록 처리
```
# (pay) Payment.java (Entity)
 @PreUpdate
    public void onPreUpdate(){
        if("OrderCancelled".equals(process)) {
            System.out.println("***** 결재 취소 중 *****");
            setProcess("PayCancelled");
            setPrice((double) 0);
            PayCancelled payCancelled = new PayCancelled();
            BeanUtils.copyProperties(this, payCancelled);
            payCancelled.publishAfterCommit();

            //Following code causes dependency to external APIs
            // it is NOT A GOOD PRACTICE. instead, Event-Policy mapping is recommended.
            phoneseller.external.Reward reward = new phoneseller.external.Reward();
            reward.setOrderId(getOrderId());
            reward.setPoint((double)-1);
            reward.setProcess("PayCancelled");
            // mappings goes here
            PayApplication.applicationContext.getBean(phoneseller.external.RewardService.class)
                    .payCancel(reward);

```
![image](https://user-images.githubusercontent.com/52647474/98260538-cb50fc00-1fc6-11eb-81f1-e681c0dde8b5.png)

- 동기식 호출이 적용되서 Reward 시스템이 장애가 나면 결재 취소처리 불가능 하다는 것을 확인:

```
#Reward 서비스를 잠시 내려놓음 (ctrl+c)

#결제취소하기(pay)
http PATCH http://localhost:8083/payments/4 price=200 process="OrderCancelled"   #Fail
```
![image](https://user-images.githubusercontent.com/52647474/98262941-88dcee80-1fc9-11eb-906a-270a7e3e68c5.png)

```
#Reward 서비스 재기동
cd reward
mvn spring-boot:run

#결제취소하기(pay)
http PATCH http://localhost:8083/payments/4 price=200 process="OrderCancelled"   #Success
```
![image](https://user-images.githubusercontent.com/52647474/98263170-d6595b80-1fc9-11eb-8b7b-cfc26a3d4f0d.png)



## 비동기식 호출 / 시간적 디커플링 / 장애격리 


결제(pay)가 이루어진 후에 Reward(store)으로 이를 알려주는 행위는 비 동기식으로 처리하여 Reward의 처리를 위하여 결제주문이 블로킹 되지 않도록 처리한다.
 
- 결제승인이 되었다(payCompleted)는 도메인 이벤트를 카프카로 송출한다(Publish)
 
![image](https://user-images.githubusercontent.com/52647474/98311688-56f07a00-2013-11eb-8e58-aff56ace7723.png)


- Reward에서는 결제승인(payCompleted) 이벤트에 대해서 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다.
- 결제완료(OrderReceive)는 송출된 결제승인(payCompleted) 정보를 reward Repository에 저장한다.:
 
![image](https://user-images.githubusercontent.com/52647474/98312150-6b814200-2014-11eb-9c1f-3df1924a67c9.png)


Reward시스템은 주문(app)/결제(pay)와 주문 프로세스 진행시 완전히 분리되어있으며(비동기 transaction 방식), 이벤트 수신에 따라 처리되기 때문에, Reward 유지보수로 인해 잠시 내려간 상태라도 주문을 받는데 문제가 없다.(시간적 디커플링):
```
# Reward 서비스를 잠시 내려놓음 (ctrl+c)

#주문하기(order)
http http://localhost:8081/orders item="GX Note99" qty=2 price=990000  #Success
![image](https://user-images.githubusercontent.com/52647474/98313378-024efe00-2017-11eb-9687-1f74803ff861.png)

#주문상태 확인

http get http://localhost:8081/orders    # 값중 Reward에서 전송해주는 point 값이 비어있고 나머지 서비스들은 정상작동하여 status가 "shipped"로 변경되어야한다.
```
![image](https://user-images.githubusercontent.com/52647474/98313550-712c5700-2017-11eb-98e6-413bb27a6cd4.png)
```
#Reward 서비스 기동
cd store
mvn spring-boot:run

#주문상태 확인
http get http://localhost:8081/orders     # 비어있었던 point 값이 입력된 것을 확인
```
![image](https://user-images.githubusercontent.com/52647474/98313747-dda75600-2017-11eb-9531-44080efb2210.png)

# 운영

## Deploy / Pipeline

- 네임스페이스 만들기
```
kubectl create ns phone82
kubectl get ns
```
![image](https://user-images.githubusercontent.com/52647474/98314504-6ffc2980-2019-11eb-9d38-852041bb7fe1.png)

- 폴더 만들기, 해당폴더로 이동
```
mkdir phone82
cd phone 82
```
![image](https://user-images.githubusercontent.com/52647474/98314552-8e622500-2019-11eb-9d2b-f3df0c728861.png)

- 소스 가져오기
```
git clone https://github.com/phone82/app.git
```

- 빌드하기
```
cd app
mvn package -Dmaven.test.skip=true
```
![image](https://user-images.githubusercontent.com/52647474/98314628-c10c1d80-2019-11eb-8820-fa7e12d263cf.png)

- 도커라이징: Azure 레지스트리에 도커 이미지 푸시하기
```
az acr build --registry admin02 --image admin22.azurecr.io/app:latest .
```
![image](https://user-images.githubusercontent.com/52647474/98314702-ea2cae00-2019-11eb-8601-9e0c71b4bd49.png)

- 컨테이너라이징: 디플로이 생성 확인
```
kubectl create deploy app --image=admin02.azurecr.io/app:latest -n phone82
kubectl get all -n phone82
```
![image](https://user-images.githubusercontent.com/73699193/98090560-83977b00-1ec7-11eb-9770-9cfe1021f0b4.png)

- 컨테이너라이징: 서비스 생성 확인
```
kubectl expose deploy app --type="ClusterIP" --port=8080 -n phone82
kubectl get all -n phone82
```
![image](https://user-images.githubusercontent.com/73699193/98090693-b80b3700-1ec7-11eb-959e-fc0ce94663aa.png)

- pay, store, customer,reward, gateway에도 동일한 작업 반복




-(별첨)deployment.yml을 사용하여 배포 

- deployment.yml 편집
```
namespace, image 설정
env 설정 (config Map) 
readiness 설정 (무정지 배포)
liveness 설정 (self-healing)
resource 설정 (autoscaling)
```
![image](https://user-images.githubusercontent.com/52647474/98315555-c9655800-201b-11eb-899f-c44d1d5a9be9.png)


- deployment.yml로 서비스 배포
```
cd app
kubectl apply -f kubernetes/deployment.yml
```


## 동기식 호출 / 서킷 브레이킹 / 장애격리

* 서킷 브레이킹 프레임워크의 선택: Spring FeignClient + Hystrix 옵션을 사용하여 구현함

시나리오는 결제(pay)->Reward 시의 연결을 RESTful Request/Response 로 연동하여 구현이 되어있고, 결제 요청이 과도할 경우 CB 를 통하여 장애격리.

- Hystrix 를 설정:  요청처리 쓰레드에서 처리시간이 610 밀리가 넘어서기 시작하여 어느정도 유지되면 CB 회로가 닫히도록 (요청을 빠르게 실패처리, 차단) 설정
```
# application.yml
feign:
  hystrix:
    enabled: true
    
hystrix:
  command:
    # 전역설정
    default:
      execution.isolation.thread.timeoutInMilliseconds: 610

```
![image](https://user-images.githubusercontent.com/52647474/98315760-590b0680-201c-11eb-8ed8-cb1d569eb256.png)
![image](https://user-images.githubusercontent.com/52647474/98318991-1bf64280-2023-11eb-9cdd-249761125ce0.png)
![image](https://user-images.githubusercontent.com/52647474/98318963-0e40bd00-2023-11eb-9f50-80dd64b378a0.png)

* siege 툴 사용법:
```
 siege가 생성되어 있지 않으면:
 kubectl run siege --image=apexacme/siege-nginx -n phone82
 siege 들어가기:
 kubectl exec -it pod/siege-5c7c46b788-4rn4r -c siege -n phone82 -- /bin/bash
 siege 종료:
 Ctrl + C -> exit
```
* 부하테스터 siege 툴을 통한 서킷 브레이커 동작 확인:
- 동시사용자 100명
- 60초 동안 실시

```
siege -c100 -t60S -r10 -v --content-type "application/json" 'http://app:8080/orders POST {"item": "abc123", "qty":3}'
```
- 부하 발생하여 CB가 발동하여 요청 실패처리하였고, 밀린 부하가 pay에서 처리되면서 다시 order를 받기 시작 

![image](https://user-images.githubusercontent.com/73699193/98098702-07eefb80-1ed2-11eb-94bf-316df4bf682b.png)

- report

![image](https://user-images.githubusercontent.com/73699193/98099047-6e741980-1ed2-11eb-9c55-6fe603e52f8b.png)

- CB 잘 적용됨을 확인


### 오토스케일 아웃

- Reward 시스템에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 15프로를 넘어서면 replica 를 10개까지 늘려준다:

```
# autocale out 설정
store > deployment.yml 설정
```
![image](https://user-images.githubusercontent.com/52647474/98317941-d769a780-2020-11eb-87be-38f875e279cd.png)

```
kubectl autoscale deploy reward --min=1 --max=10 --cpu-percent=15 -n phone82
```
![image](https://user-images.githubusercontent.com/52647474/98318004-fc5e1a80-2020-11eb-9a4b-b60a58077242.png)


-
- CB 에서 했던 방식대로 워크로드를 2분 동안 걸어준다.
```
kubectl exec -it pod/siege-5c7c46b788-4rn4r -c siege -n phone82 -- /bin/bash
siege -c100 -t120S -r10 -v --content-type "application/json" 'http://reward:8080/rewards/3 PATCH {"process":"Cancelled"}'
```
![image](https://user-images.githubusercontent.com/73699193/98102543-0d9b1000-1ed7-11eb-9cb6-91d7996fc1fd.png)

- 오토스케일이 어떻게 되고 있는지 모니터링을 걸어둔다:
```
kubectl get deploy store -w -n phone82
```
![image](https://user-images.githubusercontent.com/52647474/98318963-0e40bd00-2023-11eb-9f50-80dd64b378a0.png)

- 어느정도 시간이 흐른 후 스케일 아웃이 벌어지는 것을 확인할 수 있다. max=10 
- 부하를 줄이니 늘어난 스케일이 점점 줄어들었다.

![image](https://user-images.githubusercontent.com/52647474/98319363-f1f15000-2023-11eb-9aae-7e45e46b5f5f.png)

- 다시 부하를 주고 확인하니 Availability가 높아진 것을 확인 할 수 있었다.

![image](https://user-images.githubusercontent.com/52647474/98319633-86f44900-2024-11eb-95fe-f09e08cd420b.png)


## 무정지 재배포

* 먼저 무정지 재배포가 100% 되는 것인지 확인하기 위해서 Autoscale 이나 CB 설정을 제거함


- seige 로 배포작업 직전에 워크로드를 모니터링 함.
```
kubectl apply -f kubernetes/deployment_readiness.yml
```
- readiness 옵션이 없는 경우 배포 중 서비스 요청처리 실패

![image](https://user-images.githubusercontent.com/73699193/98105334-2a394700-1edb-11eb-9633-f5c33c5dee9f.png)


- deployment.yml에 readiness 옵션을 추가 

![image](https://user-images.githubusercontent.com/73699193/98107176-75ecf000-1edd-11eb-88df-617c870b49fb.png)

- readiness적용된 deployment.yml 적용

```
kubectl apply -f kubernetes/deployment.yml
```
- 새로운 버전의 이미지로 교체
```
cd acr
az acr build --registry admin02 --image admin02.azurecr.io/store:v4 .
kubectl set image deploy store store=admin02.azurecr.io/store:v4 -n phone82
```
- 기존 버전과 새 버전의 store pod 공존 중

![image](https://user-images.githubusercontent.com/73699193/98106161-65884580-1edc-11eb-9540-17a3c9bdebf3.png)

- Availability: 100.00 % 확인

![image](https://user-images.githubusercontent.com/73699193/98106524-c152ce80-1edc-11eb-8e0f-3731ca2f709d.png)



## Config Map

- apllication.yml 설정

* default쪽

![image](https://user-images.githubusercontent.com/73699193/98108335-1c85c080-1edf-11eb-9d0f-1f69e592bb1d.png)

* docker 쪽

![image](https://user-images.githubusercontent.com/73699193/98108645-ad5c9c00-1edf-11eb-8d54-487d2262e8af.png)

- Deployment.yml 설정

![image](https://user-images.githubusercontent.com/73699193/98108902-12b08d00-1ee0-11eb-8f8a-3a3ea82a635c.png)

- config map 생성 후 조회
```
kubectl create configmap apiurl --from-literal=url=http://pay:8080 --from-literal=fluentd-server-ip=10.xxx.xxx.xxx -n phone82
```
![image](https://user-images.githubusercontent.com/73699193/98107784-5bffdd00-1ede-11eb-8da6-82dbead0d64f.png)

- 설정한 url로 주문 호출
```
http POST http://app:8080/orders item=dfdf1 qty=21
```

![image](https://user-images.githubusercontent.com/73699193/98109319-b732cf00-1ee0-11eb-9e92-ad0e26e398ec.png)

- configmap 삭제 후 app 서비스 재시작
```
kubectl delete configmap apiurl -n phone82
kubectl get pod/app-56f677d458-5gqf2 -n phone82 -o yaml | kubectl replace --force -f-
```
![image](https://user-images.githubusercontent.com/73699193/98110005-cf571e00-1ee1-11eb-973f-2f4922f8833c.png)

- configmap 삭제된 상태에서 주문 호출   
```
http POST http://app:8080/orders item=dfdf2 qty=22
```
![image](https://user-images.githubusercontent.com/73699193/98110323-42f92b00-1ee2-11eb-90f3-fe8044085e9d.png)

![image](https://user-images.githubusercontent.com/73699193/98110445-720f9c80-1ee2-11eb-851e-adcd1f2f7851.png)

![image](https://user-images.githubusercontent.com/73699193/98110782-f4985c00-1ee2-11eb-97a7-1fed3c6b042c.png)



## Self-healing (Liveness Probe)

- store 서비스 정상 확인

![image](https://user-images.githubusercontent.com/27958588/98096336-fb1cd880-1ece-11eb-9b99-3d704cd55fd2.jpg)


- deployment.yml 에 Liveness Probe 옵션 추가
```
cd ~/phone82/store/kubernetes
vi deployment.yml

(아래 설정 변경)
livenessProbe:
	tcpSocket:
	  port: 8081
	initialDelaySeconds: 5
	periodSeconds: 5
```
![image](https://user-images.githubusercontent.com/27958588/98096375-0839c780-1ecf-11eb-85fb-00e8252aa84a.jpg)

- store pod에 liveness가 적용된 부분 확인

![image](https://user-images.githubusercontent.com/27958588/98096393-0a9c2180-1ecf-11eb-8ac5-f6048160961d.jpg)

- store 서비스의 liveness가 발동되어 13번 retry 시도 한 부분 확인

![image](https://user-images.githubusercontent.com/27958588/98096461-20a9e200-1ecf-11eb-8b02-364162baa355.jpg)

