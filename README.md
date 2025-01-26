# 네고왕 이벤트 선착순 쿠폰 시스템
# 개요
fastcampus 9개 프로젝트로 경험하는 대용량 트래픽 & 데이터 처리 초격차 패키지 Online.  
네고왕 이벤트 선착순 쿠폰 시스템

# locust 워커 스케일 아웃
docker-compose up -d --scale worker=3

# 테스트케이스
@Builder 어노테이션으로 간편하게 데이터 셋팅 후 진행. CouponTest

# QueryDslConfiguration
Java 기반 ORM(Object-Relational Mapping) 라이브러리에서 타입 안전한 동적 SQL 쿼리를 작성하기 위한 프레임워크.  
주로 JPA와 함께 사용되며, SQL 쿼리를 Java 코드로 작성할 수 있는 직관적인 API를 제공.  
SQL을 문자열로 작성하지 않고, 코드 기반으로 작성하기 때문에 컴파일 시점에 쿼리의 오류를 발견할 수 있음

# application-core.yml에 ---로 다중 환경 설정
설정 파일에 ---으로 구분하여 한 파일에 여러 환경설정 기록이 가능  
이걸 왜 몰랐지?
```yaml
spring:
  config:
    activate:
      on-profile: local
---
spring:
  config:
    activate:
      on-profile: test
---
spring:
  config:
    activate:
      on-profile: prod
```

# @ActiveProfiles("test")
테스트케이스에 위 어노테이션으로 test환경 파일 지정 가능

# @TestPropertySource(properties = "spring.config.name=application-test")
테스트에서 사용할 속성을 지정

# @SpringBootTest
Spring Boot 테스트 환경을 초기화하고, 모든 빈(bean)을 로드, 통합 테스트를 수행할 때 사용되며, 스프링 컨텍스트가 완전하게 구성됨

# 동시성 이슈 해결 방법
정해진 수량보다 과하게 발행되는 오류 발생하여 수정     
1차: @Transactional + 발급매서드에 synchronized 사용. 단일 노드에서 서비스한다는 보장이 없고 요청이 많아질 경우 스래드풀이 고갈되면서 서비스 불가가 됨.  
synchronized가 락을 반납하고 아직 커밋을 못한 상태인데 그 다음 요청이 락을 획득하면서 진입하여 또 과하게 발행됨

2차: synchronized를 더 상위 클래스의 매서드로 보내서 기존의 트랜잭션시작-락획득-처리-락반납-트랜잭션커밋 구조를 락획득-트랜잭션시작-처리-트랜잭션커밋-락반납으로 변경하여 해결  
그러나 단일 노드라면 사용가능하겠지만 역시나 다중노드에서는 사용불가한 방법

3차: 분산락 구현. redisson 락으로 해결. CouponIssueRequestService의 issueRequestV1메서드 안synchronized를 DistributeLockExecutor.excute로 대체

4차: mysql의 for update(X lock, 엑스클루시브락)으로 해결. CouponJpaRepository 참고

5차: DistributeLockExecutor를 통한 redis락을 걸지 않고  EVAL Script로 동시성 이슈 해결. AsyncCouponIssueServiceV2, RedisRepository issueRequestScript 참고

# Ch. 5부터는 발급 요청과 발급을 분리
redis의 sortedset 활용 예정 -> 생각보다 스코어 관리의 필요성이 없어(동일 시간으로 생성된 레코드 존재) 일반 set사용시보다 이점이 없어 set사용으로 변경

# Ch. 7 Redis기반 쿠폰 발급 서버 구현부터 계속

# 이하 기존 마크다운 파일 내용

---

## Background
네고왕 선착순 쿠폰 이벤트는 한정된 수량의 쿠폰을 먼저 신청한 사용자에게 제공하는 이벤트입니다.

## Requirements
- 이벤트 기간내에(ex 2023-11-03일 오후 1시 ~ 2023-11-04일 오후 1시) 발급이 가능합니다.
- 선착순 이벤트는 유저당 1번의 쿠폰 발급만 가능합니다.
- 선착순 쿠폰의 최대 쿠폰 발급 수량을 설정할 수 있어야합니다.

## Architecture

### 비동기 쿠폰 발급 요청 처리 구조
<img width="1073" alt="스크린샷 2023-10-22 오후 9 02 11" src="https://github.com/prod-j/coupon/assets/148772205/fcf94798-b60d-4776-9101-4958c655510c">

### 캐시 데이터 기반 Validation
![image](https://github.com/prod-j/coupon/assets/148772205/b374181e-5883-4cf7-98c3-9f457855b26b)

### Tech Stack

**Infra**  
Aws EC2, Aws RDS, Aws Elastic Cache,

**Server**  
Java 17, Spring Boot 3.1, Spring Mvc, JPA, QueryDsl

**Database**  
Mysql, Redis, H2

**Monitoring**   
Aws Cloud Watch, Spring Actuator, Promethous, Grafana

**Etc**  
Locust, Gradle, Docker

## Main Feature

### 쿠폰 발급 검증
- 발급 기한
- 발급 수량
- 중복 발급

### 쿠폰 발급 수량 관리
- Redis Set기반 재고 관리

### 비동기 쿠폰 발급
- Redis List (발급 Queue)
- Queue Polling Scheduler

## Result

#### server spec
- coupon-api, coupon-consumer -> t2.medium (2core, 4G)
- Mysql -> db.t3.micro (free-tier)
- Redis -> cache.t3.micro	(free-tier)

### Load Test
Runtime : 5m  
Number of Users : 5,000

#### Request Statics
Total Requests : 1,696,253
Failed Requests : 1,315
Failure rate : 0.077%   
RPS : 4958

#### Response Statics
p50 : 900 ms  
p70 : 1200 ms  
p90 : 1500 ms  
p90 : 2000 ms  
p99 : 3600 ms  
max : 14000 ms

<img width="1032" alt="image" src="https://github.com/prod-j/coupon-version-management/assets/148772205/9a1b0725-a8a9-4a86-9d49-85490351dcc8">

#### Chart
<img width="786" alt="image" src="https://github.com/prod-j/coupon-version-management/assets/148772205/9b37ab5f-7d77-41e0-9358-da0aea10a725">

#### Metric
<img width="1656" alt="image" src="https://github.com/prod-j/coupon-version-management/assets/148772205/9e3936c5-698d-40cb-b4f4-60e98993687e">
