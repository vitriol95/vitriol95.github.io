---
title: TestContainers
date: 2021-10-01 11:00:00 +0900
categories: [DevOps,test]
tags: [test,testContainers,docker]
---

## 서론

테스트 환경은 프로젝트 설정에서 작지 않은 부분을 차지하고 있고, TDD의 등장으로 그 중요성이 커지고 있는 것이 사실이다. 이번 프로젝트에서는 테스트에 있어 꽤 유명한 툴인 Test Container를 사용해보고, 이에 대한 리뷰 및 의견을 적어보려한다.

<br/>

### 왜 사용하는가?

1. 테스트 환경을 만드는 과정에서 신경써야할 부분은 여러가지가 있겠지만, 가장 중요한 요소로 '멱등성'을 꼽게된다. 이를 간과한 경우에는 다른 테스트 혹은 외부 모듈로 인해 테스트가 간헐적으로 실패할 수 있고, 구간을 찾기가 매우 힘들어진다.
> 멱등성이란 연산을 여러번 적용하더라도 결과가 바뀌지 않은 성질을 뜻한다. (HTTP메서드 중, GET PUT 등이 있겠다.)

2. 로컬 환경에서 잘 실행되던 테스트가 누군가의 환경이나 CI에서 깨지기 시작하면 이에 대해 많은 피로감과 신뢰저하를 가지게 된다.
  - 이를 고치는 데에 기존보다 훨씬 더 많은 시간과 리소스를 쏟아야하기에 생산성 저하로 이어질 수 있다.

3. 만약 다른 컴포넌트에 테스트가 의존하게 되면, 그 컴포넌트가 항상 일정한 응답을 내려줄 것이라는 가정을 하게 된다.
  - 이러한 종속성이 들어오게 되면 오히려 테스트 코드가 비즈니스 코드에 영향을 줄 수 있는 환경이 만들어 질 수있다.

<br/>

### 이러한 문제에 대한 대안은?

테스트 환경을 위해 가장 많이 쓰이는 외부 의존성을 떠올리자면 가장 먼저 DB가 떠오른다. DB를 테스트 환경에서 실행하는 방법으로는 여러가지가 존재한다.

1. Local에 실제 DB를 띄우기
  - 실제와 거의 유사한 상황에서 테스트를 할 수 있지만, 동시에 여러 테스트가 이루어지거나 테스트 이후에도 데이터가 남아있는 문제들로 인해 멱등성 관리가 어려울 수 있다.
  - 이는 특히 DDL을 테스트 하는 경우 매번 테이블에 대해 drop을 해주어야한다.

2. in-memory db사용 (h2와 같은)
  - 매우 빠르게 동작한다는 장점이 존재한다. 그리고 ORM을 통해 특정 DB에 종속성을 없앨 수 있다는 장점 또한 존재한다.
  - 하지만 특정 DB에 특화된 기능을 테스트하거나 DDL을 명시할 때, 실제 동작과는 다른 SQL을 작성해야 한다는 문제가 존재한다.

3. Docker compose
  - 도커 이미지를 통해 테스트 환경을 구축할 수 있다. 나는 개인적으로 이것도 굉장히 좋은 솔루션이라고 생각하며, 실제 TestContainer도 컴포즈파일을 지원한다!.
  - 하지만 yml파일에 대한 관리가 필요하다. 예를 들어 port를 바꾼다면 코드에서도 한번 더 port를 바꿔주어야한다.
  > 결국 random port를 통한 local포트 충돌 방지 및 parallel 테스트 환경을 조성하는데 큰 걸림돌이 될 수 있다.

4. Test Container
  - 동작 원리는 docker-compose와 큰 차이는 없다. 하지만 외부 설정 파일 없이 Java(JVM)언어만으로 docker container를 활용한 테스트 환경을 설정할 수 있다.
  - 특히 compose를 활용할 때 필요한 container와의 통신 또한 언어 레벨에서 처리가 가능하다.
  - 즉, container에 변경사항이 생기더라도 여러 곳을 변경할 필요 없이 하나의 코드로 관리할 수 있다는 장점이 존재한다.

## 테스트 컨테이너란 그래서 무엇인가

<img src="/assets/img/devOps/testcontainer/1.JPG">

- 공식 GitHub에 첨부되어 있는 설명에 해당한다. 결국 Junit을 지원하면서 도커 엔진기반의 경량의 1회용 DB를 제공한다는 것이다.
- 테스트를 위해 Docker Container를 실행시켜주는 자바 라이브러리에 해당한다.

### 사용법(Junit5 기준)

- 사용법 자체는 정말 간단하다. 도커이미지이름을 생성자로 넘긴 XXXContainer를 생성해주고, @Container 어노테이션을 달아주면 해당 DB를 도커 허브에서 가져와 사용한다.
- 그리고 해당 테스트 클래스 상단에 @TestContainer 어노테이션을 적어주어 활성화 시킨다.
> 당연히, 도커 엔진 기반으로 활동하기에 도커가 깔려있어야 한다.

- 그리고 다음과 같은 작업을 실행하게 된다.
  - 테스트 메서드 실행 이전에 activated
  - 로컬의 docker환경을 테스트
  - 이미지가 필요하다면 (처음 실행시) pull
  - 해당 이미지 베이스의 컨테이너를 실행시키고 준비
  - 테스트가 종료되면 자동으로 종료하며 컨테이너를 지운다( ps -a 명령어를 주어도 보이지 않고, images에만 남게된다)
> 자동으로 비어있는 port로 컨테이너를 실행시키게 되며, 이 포트매핑 정보를 수동으로 설정해 줄 수도 있다.

#### Ryuk Container


- 나는 처음에 mysql:5.7.34 버전을 명시하여 컨테이너를 설정해 두었는데, 설정해주지 않은 Ryuk container또한 함께 실행되고, pull되어 있었다.
- Ryuk Container는 무엇이며 왜 사용되는 것일까.

도커 허브 홈페이지에 들어가 Info를 보면 다음과 같이 서술 되어있다.
> This project helps you to remove containers/networks/volumes/images by given filter after specified delay.

아하... 결국 테스트 컨테이너에 대한 Daemon thread의 역할을 해주고 있던 것이다. Ryuk이라는 Container가 도커데몬을 이용해 테스트 컨테이너의 stop및 rm을 진행해 주는 것이다.

### 적용할 때 신경 써야 했던 것들(with Kotlin)

```kotlin
abstract class MySQLTestContainer {

    companion object {
        @Container
        val mysqlContainer = KMySQLContainer(image = "mysql:5.7.34").apply {
            withDatabaseName("message-test")
            withCommand("mysqld","--character-set-server=utf8mb4")
            withInitScript("schema.sql")
            start()
        }

        @JvmStatic
        @DynamicPropertySource
        fun datasourceConfig(properties: DynamicPropertyRegistry){
            properties.add("spring.datasource.url", mysqlContainer::getJdbcUrl)
            properties.add("spring.datasource.password", mysqlContainer::getPassword)
            properties.add("spring.datasource.username", mysqlContainer::getUsername)
        }
    }
}

class KMySQLContainer(val image: String): MySQLContainer<KMySQLContainer>(image)
```
---

#### 1. 클래스 선언

컨테이너를 생성하는 데에 있어 제네릭이 쓰이는데, 해당 생성자를 까고 들어가보면 다음과 같은 구조로 이루어져 있다. <br/>
`public class MySQLContainer<SELF extends MySQLContainer<SELF>> extends JdbcDatabaseContainer<SELF>`

<br />
자기 자신을 제네릭으로 삼아 다시 선언을 해주어야하는 recursive한 구조에 해당한다. 더 까서 들어가보면 더 괴이한(?) 구조가 나오게 된다.

`JdbcDatabaseContainer<SELF extends JdbcDatabaseContainer<SELF>> extends GenericContainer<SELF>`
<br/>
결국 근본은 GenericContainer라는 구체적인 DB구현체가 명시되지 않은 GenericContainer가 나오며, JDBC connection을 expose 해주는 JdbcDatabaseContainer로 상속을 이어주고 있다.
> MySQLContainer는 이를 상속받아 사용하는 것이다.

<br />

그럼 이 타입 파라미터인 SELF는 어디에 쓰이는것인가 보니, 아래와 같은 곳에서 사용되고 있었다.

<img src="/assets/img/devOps/testcontainer/2.png">

이걸 보니, 왜 이렇게 사용해야하는지 감이 온다. 결국 (나의 경우는 KMySQL~)새로운 클래스로 한번 더 덮어 써 줌으로써 우리가 원하는 설정들을 덮어 씌워줄 수 있는 것이다. 익숙치 않은 일종의 wrapping 이다보니, 조금 헤멨었다.


#### 2. @DynamicPropertySource

나름 최신(?) 버전에서 지원하고 있는 어노테이션에 해당한다. 본적이 거의 없어서.. 어떤 문제를 해결하려 나온 어노테이션인지 확인하는 과정이 필요했다. [여기를 참조했다](https://www.baeldung.com/spring-dynamicpropertysource)

<br>

여기서도 이 어노테이션의 사용 예시를 TestContainer를 이용해 설명하고 있다. 핵심은 이 구절에서 나온다.
> __If our PostgreSQL container is going to listen to a random port every time, then we should somehow set and change the spring.datasource.url configuration property dynamically__

이 구절을 보니 느낌이 팍 왔다. 랜덤 포트에서 실행되는 컨테이너에다가, 우리는 jdbc connection에 대한 datasource 정보를 넘겨주어야 한다.

- 만약 configuration이 static하다면, @PropertySource를 이용해 해당설정 파일에 해당하는 yml이나 properties 파일을 줄 수 있다.
- 하지만 dynamic한 경우엔 달라진다. 
  - 이를 해결하기 위해 전통적인 방식으로는 static한 custom ApplicationContextInitializer를 구현하여 제공하는 방식이 존재했다. (위 링크에 그 과정이 나와있다.)
  - 코드를 보면 알지만, 상당히 복잡하고 테스트코드 자체가 장황해진다.
- 이를 좀 더 편리하게 해주는 것이 @DynamicPropertySource에 해당한다. 내 코드와 같이 registry를 가져와서 원하는 설정들을 add(String, Supplier<Object>)에 넣어주기만 하면 된다.
> 결국 전통적인 ApplicationContextInitializer을 구현하는 방식을 어노테이션 안으로 감추어 준 것이다. 
> 그리고! 당연히 static으로 구성되어야 한다. 메서드 형태이므로 코틀린에서 지원하는 @JvmStatic 어노테이션을 붙혀주었다.

#### 3. 설정 문제
- 앞선 @DynamicPropertySouce를 이용하여 설정을 넘겨주었는데, default로 유저네임과 패스워드는 test가 되며 스키마의 이름 또한 test로 설정된다.
- 이후, 한글이 모두 깨져 테스트가 실행이 안되는 현상이 발생했었는데, 이는 mysql초기 세팅 charset에 대한 문제이다.
  - my.cnf에 직접 접근할 수 없으니, withCommand라는 설정을 제공한다. 여기서 charset을 바꾸어주면 된다
  > InitScript또한 추가해줄 수 있다!! 