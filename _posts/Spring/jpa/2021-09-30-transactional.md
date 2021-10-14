---
title: JPA의 @Transactional 어노테이션
date: 2021-09-30 14:00:00 +0900
categories: [Spring, jpa]
tags: [spring,springboot,jpa,orm,hibernate]
---

# Transactional

애플리케이션 구현을 할때, 서비스나 리포지토리 단에서 @Transactional이라는 어노테이션을 붙히는 경우가 여럿 있었다. 오늘은 이 어노테이션에 대해 파고 들어가보려한다.

<br/>

트랜잭션 격리성에 대한 이야기가 나오므로 
[이글](https://vitriol95.github.io/posts/mysql1/)을 보고오면 좋다!

## 기본 기능

Spring에서 JPA를 사용할 때, 빼놓을 수 없는 기능중 하나이다. 실제로 Spring Data Jpa를 사용하게 되었을때, 리포지토리들에게는 (ex. Crud, Jpa) 모두 `@Transactional` 이 달려있다. 이 어노테이션이 주는 기능은 크게 두가지이다.

- Transaction의 begin, commit을 자동으로 수행해준다.
- 예외가 발생하였을 때, rollback 처리를 자동으로 수행해준다.
> 여기서 말하는 예외는 Unchecked에 해당한다.

## Deep dive

첫번째 기능을 보자. JPA에서는 트랜잭션 매니저자체가 추상화되어 제공되기 때문에 JDBC를 직접 이용한다고 하였을 때 상황이다.

```java
Connection connection = dataSource.getConnection();

try (connection){
    connection.setAutoCommit(false);
    // SQL Statements ~ ...
    connection.commit();
} catch (SqlException e){
    connection.rollback();
}
```
`datasource`에 연결하고 이에 대한 커넥션을 try-with-resource 문으로 가져와 SQL문에 대해 실행을 해주는, 기본적인 JDBC를 사용할때 보는 포맷이다.

<br/>

여기서 `connection.setAutoCommit(false)`는 실제 자바에서 DB와 트랜잭션을 시작하는 유일한 방법에 해당한다. false옵션을 주게되면 Jdbc의 룰에 따르지 않고 내가 트랜잭션에 대해 제어(커밋 or 롤백)을 진행한다.

<br/>

위 방법이 JDBC의 기본적인 트랜잭션관리 방법이며 Spring의 `@Transactional`에 해당한다. 


### 옵션

`@Transactional`의 어노테이션을 사용하면 다음과 같은 옵션과 함께 사용이 가능하다. 

```java
@Transactional(propagation=TransactionDefinition.NESTED, 
isolation=TransactionDefinition.ISOLATION_READ_UNCOMMITTED
```

이와 같은 어노테이션 파라미터들도 마찬가지로 JDBC로 정리하면 다음과 같다.

```java
connection.setTransactionIsolation(Connection.TRANSACTION_READ_UNCOMMITTED); // 고립수준 설정
Savepoint save = connection.setSavePoint(); // nested 옵션
```

이처럼 내부에 들어있는 파라미터는 위와 같은 코드들을 사용자 레벨에서 간편하게 쓸 수 있도록 추상화 시켜준 것에 불과하다.

### 자동 설정

스프링을 사용한다면, Configuration에 `@EnableTransactionManagement` 어노테이션을 붙인 뒤, 트랜잭션 매니저를 빈으로 등록해 사용해야한다. 예를 들면 다음과 같은 코드이다.

```java
@Configuration
@EnableTransactionManagement
public class TxConfig{
    @Bean
    public PlatformTransactionManager txManager(){
        return myTxManager;
    }
}
```

하지만, 스프링 부트 사용시 `@EnableTransactionManagement`는 자동으로 처리해주며 트랜잭션 매니저또한 자동 주입이 진행된다. 이를 사용하려면 `@Transactional` 어노테이션만 선언해주면 된다.
> Transaction Manager와, EntityManager는 다르다!!

만약 어떠한 서비스에 `@Transactional`이 붙는다면 실제 코드는 어떻게 바뀌는지 확인해 보자.

```java
public class MyService{

    @Transactional
    public Account signUp(Account account){
        // service code ...
        return accountRepository.save(account)
    }
}
```
이는 JDBC로 다음과 같이 변환될 것이다.

```java
public class MyService{

    public Account signUp(Account account){
        Connection connection = dataSource.getConnection();
        try(connection){
            connection.setAutoCommit(false);
            // SQL statements ...
            connection.commit();
        } catch(SQLException e){
            connection.rollback();
        }
    }
}
```

이쯤되면 슬슬 감이온다. 우리가 실제로 쓰는 서비스 코드들은 저 SQL statements에 말려 들어갈 것이며 실제 `@Transactional`은 트랜잭션에 대한 설정 및 커밋, 롤백만 진행해주는 것이다.

### 구현 방식

그렇다면, 스프링은 어떻게 `@Transactional`만으로 저 구현을 도와줄까? 어느정도 예상할 수 있듯이, `프록시`를 활용하고 있다. 즉, 앞선 코드의 MyService를 인스턴스화 할뿐만 아니라, 해당 클래스를 상속한 프록시도 인스턴스화 하게 된다.
> 아래와 같은 과정을 거친다고 보면 되겠다.

1. 스프링이 `@Transactional` 어노테이션을 발견하면 해당 빈의 다이나믹 프록시를 만든다. (By CGLIB)
2. 해당 프록시 객체는 Tx 매니저에 접근하고 트랜잭션이나 커넥션의 여닫기를 요청한다.
3. 트랜잭션 매니저는 JDBC방식으로 코드를 실행해 줄 것이다.

Pseudo Code를 작성해 보자면 아래와 같이 행동한다는 것이다.

```java
public class MySerivceProxy extends MyService {

    private final TransactionManager manager = TransactionManager.getInstance();

    public Account signUp(Account account){
        try {
            manager.begin();
            // 기존 작성된 서비스 코드 ~~
            // ...

            manager.commit();
        } catch(Exception e){
            manager.rollback();
        }
    }

}
```
위임할 객체가 있다는 것을 제외하면 스프링 빈이 프록시로 생성되어 싱글턴으로 관리되며 작동하는 방식과 유사하다.

# Transactional의 파라미터들

그렇다면 이 `@Transactional` 과 함께 사용할 수 있는 파라미터들은 어떤 것들이 있는지 확인해보자.

## Propagation Level

말 그대로 트랜잭션의 전파 레벨을 명시하는 것에 해당한다. 다음과 같은 옵션들이 존재한다.

- Required(Default): 트랜잭션을 새로 열거나, 기존에 있는 것을 사용
> setAutoCommit(false)

- Supports: 트랜잭션에 대해 아무것도 손대지 않는다. (JDBC는 어떠한 행동도 하지 않는다)

- Mandatory: 다른 트랜잭션에 의존한다. 즉, 트랜잭션이 열려있어야 한다. (마찬가지로 JDBC는 아무런 행동도 하지 않는다)

- Required_new: 해당 요청에 대해 새로운 독립된 트랜잭션을 열어 사용한다.

- Nested: 저장점을 잡아준다(SavePoint를 설정해준다)

## Isolation Level

트랜잭션 격리성에 대한 설정에 해당한다.

- DEFAULT: 기본값에 해당하며 해당 DB의 Isolation Level을 따라가게 된다.
> MySQl의 경우는 REPEATABLE READ를 따라가게 될 것이다.

- READ_UNCOMMITED
- READ_COMMITED (보통 다른 DB들의 기본값에 해당한다.)
- REPEATABLE_READ
- SERIALIZABLE

마찬가지로 격리성에 대한 설명은 [이글](https://vitriol95.github.io/posts/mysql3/)에 잘 나와있다.

## rollbackFor

해당 예외에 대해 강제로 Rollback 시키는 옵션이다. 그냥 `@Transactional`만 달랑 적어두게 되면 UncheckedException에 대해서만 롤백을 진행하게 되는데, CheckedException들에 대해서도 해당 옵션을 주면 롤백을 진행한다.

## noRollbackFor

해당 예외에 대해서는 롤백을 하지않고 진행한다.

## timeout

지정한 시간 내에 해당 작업이 수행완료되지 않는다면 rollback을 수행하게 된다. Default값은 -1로, 이는 no timeout을 의미한다.

## readOnly

읽기 전용 트랜잭션을 열게 된다. 이는 Persistence provider에게 주는 일종의 힌트로, `읽기만 할게요`라는 것을 줌으로써 성능상의 이점을 가져올 수 있다. 이 옵션은 나도 굉장히 많이 사용하는데 이에 대해서는 좀 더 알아보자. 

<br/>

기본적으로 쿼리 진행도중, Hibernate는 Update나 Delete 전후에 `Flush()` 를 진행하여 데이터베이스에 sync작업을 하게 된다. 
> 보통은 매니저가 적절하게 배치하게 되며, 데이터를 read해오기전에 flush요청이 날아가게 된다.

하지만 `readOnly`옵션을 `true`로 줄 경우 Flush모드를 Never로 설정하게 되어, 이로부터 오는 오버헤드들을 모두 피할 수 있게 된다. (Dirty Checking또한 작동하지 않는다.)

<br/>

데이터 변경이 일어나지 않을 트랜잭션이기 때문에, 영속성 컨텍스트에 해당 엔티티들의 영속화를 진행하지 않으며 Flush를 하지 않아 데이터베이스 동기화도 진행하지 않는다.
> 따라서 읽어오는 작업인데, 데이터 양이 많다면 이 옵션을 사용하면 성능 향상을 기대할 수 있다.

실제 Javadoc을 살펴보면 readOnly는 다음과 같은 정의를 가지고 있다.

```text
This just serves as a hint for the actual transaction subsystem 
it will not necessarily cause failure of write access attempts. 
A transaction manager which cannot interpret the read-only hint 
will not throw an exception when asked for a read-only transaction.
```

여기서는 일종의 pitfall을 소개하고 있는데, 이는 다음과 같은 경우가 있겠다.

```java
@Transactional( propagation = Propagation.SUPPORTS,readOnly = true )
```
해당 propagation 옵션을 주면 JDBC는 트랜잭션에 대해 손대지 않기에 실제로 이 과정이 읽기로만 이루어져 있는지, 기대하는 대로 작동하는지에 대해 확신을 주지 못한다는 것이다.

# JPA + Hibernate

자 이제, 좀 더 내 관점으로 들어와서 이야기를 해보자. 스프링에서 JPA를 사용한다면 구현체로 Hibernate를 이용하게 되는데, 이때는 어떻게 동기화가 진행될까?
> 놀랍게도 스프링과 하이버네이트의 통합은 이미 이루어져 있다.

> 어차피 두 트랜잭션은 모두 유일한 방법인 JDBC의 기본 방식을 사용하기 때문이다.

트랜잭션 매니저를 빈으로 사용할 경우에는 JDBC를 사용했을 때 `DataSourcePlatformTransactionManager`를 쓰는 대신에, `HibernateTransactionManager`를 사용한다면 이 동기화는 자동으로 이루어진다. JPA를 통해 Hibernate를 사용한다면 `JpaTransactionManager`를 쓰면 되는 것이다.

<br/>

`JpaTransactionManager`는 JPA를 통해서 하이버네이트를 사용할 때 이에 대한 트랜잭션을 관리해준다. `spring-boot-starter-jpa`의존성을 가져오게 되면 자동으로 `JpaTransactionManager`를 사용하게 된다.

## 결론

하이버네이트가 되었든 JPA가 되었든 스프링의 @Transactional을 활용을 하든지 간에, JDBC의 기본 방식으로 결국 접근을 한다.
> getConnection().setAutoCommit(false).commit()

# 출처

- [기본기를 쌓는 정아마추어 코딩블로그](https://jeong-pro.tistory.com/228)
- [Transaction-config-with-jpa-and-spring](https://www.baeldung.com/transaction-configuration-with-jpa-and-spring)
- [프리라이프의 기술 블로그](https://freedeveloper.tistory.com/159)