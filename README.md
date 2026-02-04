# Warning
<span style="color:red;">해당 프로젝트는 삭제 또는 리팩토링할 예정입니다. (2026-02-04)</span>
---
# 프로젝트 소개

## 이름

다중 금융상품 알고리즘 투자 알림 봇

## 설명

`가상화폐 거래소`와 `증권사`의 Open API를 사용해서 금융상품 가격을 조회하고, 지표 분석을 통해 주문 시점을 알려주는 프로그램이다.  
현재, 사용할 수 있는 거래소는 `ByBit(가상화폐 거래소)`과 `LS 투자증권(증권사)`이다.

## 요구사항

- Open API를 제공하는 다른 `가상화폐 거래소`와 `증권사`를 추가할 수 있어야 한다
- 라이브러리와 프레임워크를 쉽게 변경할 수 있어야 한다

## 목적

- TDD 훈련
- 핵사고날 아키텍처 실험
- Kotlin 과 SpringBoot WebFlux 학습

## 사용한 기술

- 프로그래밍 언어
    - [Kotlin](https://kotlinlang.org/)
- 프레임워크
    - [Spring Boot WebFlux](https://docs.spring.io/spring-framework/reference/web/webflux.html) (coroutine 기반): 웹 애플리케이션
      조립에 사용
    - [Spring Data R2DBC](https://spring.io/projects/spring-data-r2dbc): 영속성 데이터 접근에 사용
- 라이브러리
    - [OkHttp](https://square.github.io/okhttp/): http 통신, websocket 통신에 사용
    - [jackson](https://github.com/FasterXML/jackson): JSON 데이터 변환에 사용
    - [Jakarta Bean Validation](https://beanvalidation.org/): 입력 유효성 검사에 사용
    - [ta4j](https://github.com/ta4j/ta4j): 캔들 데이터 관리와 지표 생성에 사용
    - [Liquibase](https://www.liquibase.com/): RDB 스키마 관리를 위해 사용
    - [Testcontiners](https://testcontainers.com/): RDB 테스트에 사용

---

# 문제 해결

## TestContainer 와 Liquibase 를 사용한 RDB 테스트 환경 구성

아래와 같이 `RdbTestContainer` 클래스에서 TestContainer 를 전역적(companion object)으로 설정해서 사용한다.   
시스템 프로퍼티(`X_DBMS_NAME`)로 RDBMS 를 변경할 수 있다. (Postgresql, MySQL 지원)

https://github.com/newy2/algo-trade-backend/blob/dc1d97db173090985ef716a75364a795136a4e85/api-server/web-flux/src/test/kotlin/helpers/spring/RdbTestContainer.kt#L12-L38

## RDB 테스트 코드에서 자동 롤백 기능 사용하기

Spring Data R2DBC 에서는 테스트 메서드에 `@Transactional` 애너테이션을 사용한 자동 롤백 기능을 지원하지
않는다. ([관련 이슈](https://github.com/spring-projects/spring-framework/issues/24226))  
해당 프로젝트에서는 `BaseDataR2dbcTest#runTransactional` 메서드로 자동 롤백 기능을 사용한다.  
(`BaseDataR2dbcTest` 는 `RdbTestContainer` 를 상속한다)

https://github.com/newy2/algo-trade-backend/blob/dc1d97db173090985ef716a75364a795136a4e85/api-server/web-flux/src/test/kotlin/helpers/spring/BaseDataR2dbcTest.kt#L22-L32

`BaseDataR2dbcTest#runTransactional` 를 사용하는 테스트 코드는 아래와 같다.

https://github.com/newy2/algo-trade-backend/blob/dc1d97db173090985ef716a75364a795136a4e85/api-server/web-flux/src/test/kotlin/com/newy/algotrade/study/spring/r2dbc/AuditingTest.kt#L45-L53

## Service 컴포넌트에서 트렌젝션 커밋 이후에 로직 실행하기 (Transaction hook 테스트)

해당 프로젝트에서는 구현 편의상 Service 컴포넌트에 `@Transactional` 애너테이션을 붙여서 사용한다.  
`@Transactional` 애너테이션을 사용하면 Service 로직을 전부 실행한 이후에 DB 트렌젝션을 커밋한다.

아래와 같이 `useTransactionHook` 메서드를 사용하면 DB 변경과 직접적인 관련이 없는 로직(예: 이벤트 전송 등)을 DB 트랜젝션 커밋 이후에 호출할 수 있다.

https://github.com/newy2/algo-trade-backend/blob/8e4b590b3e382fe6a79feef75d1fa24fdb71b2d5/api-server/web-flux/src/main/kotlin/com/newy/algotrade/notification_app/service/SendNotificationAppVerifyCodeCommandService.kt#L20-L39

`useTransactionHook` 을 사용하는 Service 로직은 아래와 같은 순서로 테스트 한다.

1. 테스트 코드에서 `TransactionalOperator` 으로 부모 Transaction 을 오픈한다.
2. 메서드 호출 순서를 문자열 log 로 기록한다.
3. 문자열 log 를 비교하여, 부모 Transaction 커밋 이후에 해당 로직이 호출됐는지 확인한다.

https://github.com/newy2/algo-trade-backend/blob/dc1d97db173090985ef716a75364a795136a4e85/api-server/web-flux/src/test/kotlin/com/newy/algotrade/integration/notification_app/service/SendNotificationAppVerifyCodeCommandServiceTest.kt#L30-L56

`useTransactionHook` 구현 코드는 아래와 같다.  
Service 컴포넌트는 `유닛 테스트`에서도 사용하기 때문에 `forCurrentTransaction` 에 대한 예외 처리 로직을 추가한다.

https://github.com/newy2/algo-trade-backend/blob/dc1d97db173090985ef716a75364a795136a4e85/api-server/web-flux/src/main/kotlin/com/newy/algotrade/spring/hook/TransactionHook.kt#L9-L34

## IP 주소 기반 인증처리

해당 프로젝트는 다중 사용자를 지원하는 서비스가 아닌, 1인 사용자를 위한 서비스이다.  
추후 `Spring Security` 를 사용한 인증 로직을 구현할 예정이지만, 임시로 `WebFilter` 를 확장한 `AdminIpFilter` 를 구현해서, 사이트에 접속한 `IP 주소` 기반으로 사용자 인증
로직을 대체한다.

Controller 컴포넌트에서 `@AdminOnly` 애너테이션으로 관리자만 호출할 수 있는 API 를 구분한다.  
`@AdminOnly` 애너테이션의 사용법은 아래와 같다.

https://github.com/newy2/algo-trade-backend/blob/837426862b97209ad4acabe9a26d10af0ae6aa13/api-server/web-flux/src/test/kotlin/com/newy/algotrade/study/spring/auth/filter/AdminIpFilterTest.kt#L18-L64

Controller 컴포넌트 메서드의 파라미터에 `@LoginUserInfo` 애너테이션으로 사용자의 권한을 확인한다.  
`@LoginUserInfo` 애너테이션의 사용법은 아래와 같다.

https://github.com/newy2/algo-trade-backend/blob/837426862b97209ad4acabe9a26d10af0ae6aa13/api-server/web-flux/src/test/kotlin/com/newy/algotrade/study/spring/auth/filter/AdminIpFilterTest.kt#L116-L162

`AdminIpFilter` 구현 코드는 아래와 같다.

https://github.com/newy2/algo-trade-backend/blob/837426862b97209ad4acabe9a26d10af0ae6aa13/api-server/web-flux/src/main/kotlin/com/newy/algotrade/spring/auth/filter/AdminIpFilter.kt#L14-L66

`@LoginUserInfo` 애너테이션을 사용한 데이터 바인딩은 `HandlerMethodArgumentResolver`를 확장해서 구현한다.

https://github.com/newy2/algo-trade-backend/blob/837426862b97209ad4acabe9a26d10af0ae6aa13/api-server/web-flux/src/main/kotlin/com/newy/algotrade/spring/auth/resolver/AdminUserArgumentResolver.kt#L11-L23

https://github.com/newy2/algo-trade-backend/blob/837426862b97209ad4acabe9a26d10af0ae6aa13/api-server/web-flux/src/main/kotlin/com/newy/algotrade/spring/auth/config/ArgumentResolverConfig.kt#L8-L14

## Spring Data R2DBC 에서 SSL 을 사용하여 RDS(PostgreSQL 16) 에 연결하기

RDS(PostgreSQL 16)은 기본적으로 SSL 모드가 켜져 있다.  
Spring Data R2DBC 에서 SSL 을 사용하여 RDS 에 연결하기 위해서, 아래와 같은 URL 과 AWS 에서 제공하는 공개키를 사용한다.

https://github.com/newy2/algo-trade-backend/blob/dc1d97db173090985ef716a75364a795136a4e85/api-server/web-flux/src/main/resources/application.properties#L7-L8

https://github.com/newy2/algo-trade-backend/blob/dc1d97db173090985ef716a75364a795136a4e85/api-server/web-flux/src/main/resources/aws/rds/ssl/ap-northeast-2-bundle.pem#L1-L17

참고 URL:

- https://stackoverflow.com/questions/76899023/rds-while-connection-error-no-pg-hba-conf-entry-for-host#answer-78269214
- https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/PostgreSQL.Concepts.General.SSL.html

## 운영 환경별(테스트, 프로덕션, 로컬) RDBMS 의 Schema 생성 로직 추가

해당 프로젝트는 1개의 RDS 인스턴스를 사용한다. (AWS 프리티어 계정 사용)  
1개의 RDS 인스턴스에 `테스트 서버용 DB`와 `프로덕션 서버용 DB`를 다른 Schema 로 분리해서 사용한다.

아래와 같이 Liquibase 에서 `com.nocwriter.runsql` gradle 플러그인을 사용하여,   
운영 환경에 맞는 Schema 를 먼저 생성하고, Liquibase 의 ChangeLog 로 테이블을 생성한다.

https://github.com/newy2/algo-trade-backend/blob/dc1d97db173090985ef716a75364a795136a4e85/ddl/liquibase/build.gradle.kts#L86-L100

https://github.com/newy2/algo-trade-backend/blob/dc1d97db173090985ef716a75364a795136a4e85/ddl/liquibase/build.gradle.kts#L36-L51

## MySQL 의 CHAR(1) 타입이 Kotlin 의 Char 타입으로 매핑되지 않는 현상

PostgreSQL 의 CHAR(1) 타입은 Kotlin 의 Char 타입으로 매핑할 수 있지만,  
MySQL 의 CHAR(1) 타입은 Kotlin 의 Char 타입으로 매핑할 수 없다. (`io.asyncer:r2dbc-mysql` 드라이버 사용)

해당 프로젝트에서는 RDBMS의 `CHAR(1)` 타입을 Kotlin 의 `String` 타입으로 매핑해서 사용한다.

매핑 에러 테스트 코드는 아래와 같다.

https://github.com/newy2/algo-trade-backend/blob/dc1d97db173090985ef716a75364a795136a4e85/api-server/web-flux/src/test/kotlin/com/newy/algotrade/study/spring/r2dbc/MySqlDataTypeTest.kt#L41-L70

## MySQL 의 DATETIME 타입을 Kotlin 의 LocalDateTime 타입으로 매핑 시, Fractional Seconds(분수 초)가 나오지 않는 현상

PostgreSQL 의 TIMESTAMPE 타입을 LocalDateTime 타입으로 매핑하면 분수 초까지 나오지만,  
MySQL 의 DATETIME 타입을 LocalDateTime 타입으로 매핑하면 분수 초가 0으로 나오는 현상이 발생했다. (초 단위로 반올림됨)

Liquibase 에서 `전역 property` 로 날짜 타입과 기본값을 선언하고, 해당 property 를 사용해서 RDBMS 에 맞는 테이블을 생성한다.

전역 property 를 선언하는 코드는 아래와 같다.
https://github.com/newy2/algo-trade-backend/blob/dc1d97db173090985ef716a75364a795136a4e85/ddl/liquibase/master_change_log.xml#L8-L11

전역 property 를 사용하는 코드는 아래와 같다.
https://github.com/newy2/algo-trade-backend/blob/dc1d97db173090985ef716a75364a795136a4e85/ddl/liquibase/schema/algo-trade/001-tables/001_market.xml#L38-L44

참고 URL:

- https://dev.mysql.com/doc/refman/8.4/en/fractional-seconds.html

---

# 프로젝트 모듈 설명

<img src="document/image/folder-structure/project.png" width="300">

1. `api-server/webflux`
    - SpringBoot WebFlux 를 사용한 웹 애플리케이션 모듈이다
    - 웹 애플리케이션 조립과 영속성 어댑터 구현을 담당한다
2. `ddl/liquibase`
    - 개발/테스트/운영용 RDB 스키마를 관리하기 위한 모듈이다
    - [Liquibase](https://www.liquibase.com/) 를 사용해서 구현했고, 환경 변수(X_DBMS_NAME)로 `postgresql`과 `mysql`을 선택할 수 있게 구현했다

# 테스트 패키지 설명

<img src="document/image/folder-structure/test.png" width="300">

- `com.newy.algotrade.integration`
    - 외부 API 통신, RDB 접근 등 외부 프로세스와 통신하는 테스트 코드를 작성한다
- `com.newy.algotrade.study`
    - 프로그래밍 언어와 라이브러리의 사용법에 대한 테스트 코드를 작성한다
    - 프로젝트에서 사용해야 하는 문법과 라이브러리 사용법을 학습한다
    - 프로그래밍 언어와 라이브러리의 버전 업그레이드를 대비한다
- `com.newy.algotrade.unit`
    - 외부 프로세스와 통신이 필요 없는 유닛 테스트 코드를 작성한다
- `helpers`
    - 테스트 프로젝트에서만 사용하는 공통 코드를 작성한다

## 테스트 패키지를 분리한 이유

상황에 맞는 테스트 실행 명령어를 호출하기 위해서, 위와 같이 테스트 패키지 구조를 분리했다.

```bash
# 유닛 테스트 실행 명령어
./gradlew test --tests "com.newy.algotrade.unit.*" --tests "com.newy.algotrade.study.*"

# DB 테스트 실행 명령어
./gradlew api-server:web-flux:test --tests "com.newy.algotrade.integration.*"

# 전체 테스트 실행 명령어
./gradlew test
```

# webflux 모듈 패키지 설명

<img src="document/image/folder-structure/webflux.png">

- `common`: 모듈에서 공통으로 사용하는 코드 구현
- `config`: 스프링 config 코드 선언 (webflux 모듈에서 사용)
- `도메인 이름 패키지` (ex: product_price)
    - `adapter.in`: 인커밍 어댑터 구현
    - `adapter.in.web`: 웹 어댑터 구현
    - `adapter.in.web.model`: 웹 입/출력 모델 구현
    - `adapter.in.internal_system`: 내부 시스템 입력 어댑터 구현 (내부 이벤트 수신, 콜벡 port 구현 로직 등)
    - `adapter.out`: 아웃고잉 어댑터 구현
    - `adapter.out.persistence`: 영속성 어댑터 구현
    - `adapter.out.persistence.repository`: Spring Data R2dbcRepository 관련 로직 구현
    - `adapter.out.volatile_storage`: 메모리 캐시 어댑터 구현
    - `adapter.out.event_publisher`: 이벤트 발행 어댑터 구현
    - `adapter.out.external_system`: 외부 시스템 통신 어댑터 구현
    - `port.in`: 인커밍 포트 선언
    - `port.in.model`: 인커밍 포트 입력 모델 구현(입력 데이터 유효성 검증 담당)
    - `port.out`: 아웃고잉 포트 선언
    - `service`: 유스케이스 구현

---

# 프로젝트 실행 명령어

## 테스트 실행 명령어

### 유닛 테스트 실행

```bash
./gradlew test --tests "com.newy.algotrade.unit.*" --tests "com.newy.algotrade.study.*"
```

### DB 테스트 실행

```bash
# DB 테스트 실행 템플릿
./gradlew api-server:web-flux:test --tests "com.newy.algotrade.integration.*" -DX_DBMS_NAME={postgresql | mysql}

# 예시: PostgreSQL 기반 DB 테스트 실행
./gradlew api-server:web-flux:test --tests "com.newy.algotrade.integration.*" -DX_DBMS_NAME=postgresql

# 예시: MySQL 기반 DB 테스트 실행
./gradlew api-server:web-flux:test --tests "com.newy.algotrade.integration.*" -DX_DBMS_NAME=mysql
```

### 통합 테스트 실행 (ByBit, LS증권의 Open API Key 를 발급 받지 않은 경우 )

```bash
```bash
# 통합 실행 템플릿
./gradlew test -DX_DBMS_NAME={postgresql | mysql}

# 예시: PostgreSQL 기반 통합 테스트 실행
./gradlew test -DX_DBMS_NAME=postgresql

# 예시: MySQL 기반 통합 테스트 실행
./gradlew test -DX_DBMS_NAME=mysql
```

### 통합 테스트 실행 (ByBit, LS증권의 Open API Key 를 발급 받은 경우 )

```bash
# 통합 실행 템플릿
./gradlew test \
-DX_DBMS_NAME={postgresql | mysql} \
-DX_BY_BIT_API_KEY={ByBit API SECRET} \
-DX_BY_BIT_API_SECRET={ByBit API SECRET} \
-DX_LS_SEC_API_KEY={LS 증권 API KEY} \
-DX_LS_SEC_API_SECRET={LS 증권 API SECRET}

# 예시: PostgreSQL 기반 통합 테스트 실행
./gradlew test \
-DX_DBMS_NAME=postgresql \
-DX_BY_BIT_API_KEY={ByBit API SECRET} \
-DX_BY_BIT_API_SECRET={ByBit API SECRET} \
-DX_LS_SEC_API_KEY={LS 증권 API KEY} \
-DX_LS_SEC_API_SECRET={LS 증권 API SECRET}

# 예시: MySQL 기반 통합 테스트 실행
./gradlew test \
-DX_DBMS_NAME=mysql \
-DX_BY_BIT_API_KEY={ByBit API SECRET} \
-DX_BY_BIT_API_SECRET={ByBit API SECRET} \
-DX_LS_SEC_API_KEY={LS 증권 API KEY} \
-DX_LS_SEC_API_SECRET={LS 증권 API SECRET}
```

## RDB 스키마 생성, 삭제 명령어

### 스키마 생성

```bash
# DB 스키마 생성 커맨드 템플릿
./gradlew :ddl:liquibase:update \
-DX_DBMS_NAME={postgresql | mysql} \
-DX_MYSQL_JDBC_URL={MySQL JDBC URL} \
-DX_MYSQL_USERNAME={MySQL username} \
-DX_MYSQL_PASSWORD={MySQL password} \
-DX_POSTGRESQL_JDBC_URL={PostgreSQL JDBC URL} \
-DX_POSTGRESQL_USERNAME={PostgreSQL username} \
-DX_POSTGRESQL_PASSWORD={PostgreSQL password} \
-DX_LS_SEC_API_KEY={LS 증권 API KEY} \
-DX_LS_SEC_API_SECRET={LS 증권 API SECRET}

# 예시: PostgreSQL DB 스키마 생성
./gradlew :ddl:liquibase:update \
-DX_DBMS_NAME=postgresql \
-DX_POSTGRESQL_JDBC_URL=jdbc:postgresql://localhost:5432/ \
-DX_POSTGRESQL_PASSWORD=root \
-DX_POSTGRESQL_USERNAME=postgres \
-DX_LS_SEC_API_KEY={LS 증권 API KEY} \
-DX_LS_SEC_API_SECRET={LS 증권 API SECRET}

# 예시: MySQL DB 스키마 생성
./gradlew :ddl:liquibase:update \
-DX_DBMS_NAME=mysql \
-DX_MYSQL_JDBC_URL=jdbc:mysql://localhost:3306 \
-DX_MYSQL_PASSWORD=root \
-DX_MYSQL_USERNAME=root \
-DX_LS_SEC_API_KEY={LS 증권 API KEY} \
-DX_LS_SEC_API_SECRET={LS 증권 API SECRET}
```

### 스키마 삭제

```bash
# DB 스키마 생성 커맨드 템플릿
./gradlew :ddl:liquibase:dropAll \
-DX_DBMS_NAME={postgresql | mysql} \
-DX_MYSQL_JDBC_URL={MySQL JDBC URL} \
-DX_MYSQL_USERNAME={MySQL username} \
-DX_MYSQL_PASSWORD={MySQL password} \
-DX_POSTGRESQL_JDBC_URL={PostgreSQL JDBC URL} \
-DX_POSTGRESQL_USERNAME={PostgreSQL username} \
-DX_POSTGRESQL_PASSWORD={PostgreSQL password}

# 예시: PostgreSQL DB 스키마 삭제
./gradlew :ddl:liquibase:dropAll \
-DX_DBMS_NAME=postgresql \
-DX_POSTGRESQL_JDBC_URL=jdbc:postgresql://localhost:5432/ \
-DX_POSTGRESQL_PASSWORD=root \
-DX_POSTGRESQL_USERNAME=postgres

# 예시: MySQL DB 스키마 삭제
./gradlew :ddl:liquibase:dropAll \
-DX_DBMS_NAME=mysql \
-DX_MYSQL_JDBC_URL=jdbc:mysql://localhost:3306 \
-DX_MYSQL_PASSWORD=root \
-DX_MYSQL_USERNAME=root
```

---

# 개발환경 설정 & 실행 방법

```
작성 예정
```

---

# 다이어그램

## ERD

![ERD](./document/image/ERD.png)

## 클래스 다이어그램

### 캔들 차트 지표 계산 도메인

![캔들 차트](./document/image/class-diagram/candle.png)

### 그 외 클래스 다이어그램

```kotlin
작성 예정
```
