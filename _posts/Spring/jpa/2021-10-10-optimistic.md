---
title: JPA와 낙관적 락
date: 2021-09-30 14:00:00 +0900
categories: [Spring, jpa]
tags: [spring,springboot,jpa,orm,hibernate,lock,kotlin]
---

# JPA의 낙관적 락

## 개념

낙관적 락에 대한 설명은 [이글](https://vitriol95.github.io/posts/mysql4/)에 정리를 해 두었다.! 

<br/>

동시성제어를 위해 JPA에서는 낙관적 락을 지원하게 된다. 앞선 글에서와 마찬가지로 version 칼럼을 추가하여 Entity의 변경을 추적하는 메커니즘에 해당한다.

## 적용 -1 Version

JPA에서 낙관적 락을 사용하기 위해서는 `@Entity`내부에 `@Version` 어노테이션이 붙은 변수를 구현해 줌으로써 가능하다. 이때 다음과 같은 주의사항이 존재한다.

- 각 엔티티 클래스에는 `@Version`속성이 하나만 있어야 한다.
- 여러 테이블에 매핑된 엔티티의 경우, 기본 테이블에 존재해야한다.
- 타입은 int, Integer, long, Long, short, Short가 되며, Timestamp도 가능하다.
> 코틀린의 경우 맘편하게 int로 설정하면 될것이다.

JPA는 select가 일어날 때, 버전 속성값을 가지고 있다가 트랜잭션이 업데이트를 치기 전에 버전 속성을 다시 확인한다. 이 과정에서 버전 정보가 변경된다면 `OptimisticLockException`이 발생하게 된다.

## 적용 - 2 Versionless

별도의 `@Version` 없이 적용하는 것도 가능한데, 엔티티 클래스 상단에 `@OptimisticLocking`을 적용하는 방식이다. 이 어노테이션에 적용할 수 있는 4가지 모드가 있는데, 이들은 다음과 같다.

- NONE: 사용하지 않는것과 같다.
- VERSION: 적용 1의 방식을 따라간다. 별도의 version column이 필요하다.
- ALL: 모든 칼럼을 이용해 version 체크를 진행한다. update와 delete의 경우 where절 안에 모든 칼럼을 넣어 검색을 진행한다.
- DIRTY: dirty attribute(바뀐 attribute)만을 이용해 검색한다. 역시 WHERE절에 바뀐 칼럼들을 검색어로 넣어 체크한다.

예시로 보면 다음과 같은 엔티티가 나올 수 있다. (자바)

```java
@Entity(name = "Person")
@OptimisticLocking(type = OptimisticLockType.ALL)
@DynamicUpdate
public static class Person {

 @Id
 private Long id;

 @Column(name = "`name`")
 private String name;

 private String country;

 private String city;

 @Column(name = "created_on")
 private Timestamp createdOn;

 //Getters and setters are omitted for brevity
}
```

## 실습해보기 - 준비

코틀린 1.5.31, Jdk 11버전으로 진행할 생각이다. 의존성은 data jpa, h2인메모리 db를 사용하며 테스트코드는 junit5를 이용해 작성하겠다.
> 그리고 `@Version`을 적용한 버전 필드가 있는 엔티티를 사용할 생각이다. (적용1)

### Configuration & Entity


```groovy
plugins {
    // ...
    kotlin("plugin.spring") version "1.5.31"
    kotlin("plugin.jpa") version "1.5.31"
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("com.fasterxml.jackson.module:jackson-module-kotlin")
    implementation("org.jetbrains.kotlin:kotlin-reflect")
    implementation("org.jetbrains.kotlin:kotlin-stdlib-jdk8")
    runtimeOnly("com.h2database:h2")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}
// ...
```

```kotlin
@Entity
@DynamicUpdate
class TestEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    var id: Long? = null

    var count: Long = 0
    var isActive: Boolean = false

    @Version
    var version: Long = 1

    fun updateActive(){
        this.isActive = !this.isActive
    }

    fun countUp(){
        this.count++
    }

    // equals and hashcode...
}
```
> 여기서 중요한게, 버전에 대한 정보는 꼭 저렇게 Initializing 해주어야 한다.

### Service

```kotlin
@Service
@Transactional
class TestService(private val repository: TestEntityRepository) {

    private val log = LoggerFactory.getLogger(javaClass)

    fun save(){
        repository.save(TestEntity())
        log.info("=== 저장 완료 ===")
    }

    fun updateActive(){
        val entity = repository.findEntityById(1)
        entity.updateActive()
        log.info("=== 업데이트 완료 ===")
    }

    fun countUp(){
        val entity = repository.findEntityById(1)
        entity.countUp()
        log.info("=== 카운트 업 완료 ===")
    }
}
```

1. `updateActive()`는 isActive 칼럼을 뒤집는 요청에 해당하며
2. `countUp()`은 count칼럼을 1증가 시키는 요청에 해당한다.

### Repository

위에서 작성한 `findEntityById`는 다음과 같이 구현하였다.

```kotlin
interface TestEntityRepository : JpaRepository<TestEntity, Long> {

    @Lock(LockModeType.OPTIMISTIC)
    fun findEntityById(id:Long): TestEntity
}
```

실제로, JPA에서 제공하는 낙관적 Lock은 다음과 같은 4가지가 존재한다.

- OPTIMISTIC
- OPTIMISTIC_FORCE_INCREMENT
- READ (OPTIMISTIC과 동일)
- WRITE (OPTIMISTIC_FORCE_INCREMENT와 동일)

확인하였다 싶이, 가장 첫번째 부터 적용해가며 사용할 것이다.

## 실습해보기 - 확인

### LOCKTYPE - OPTIMISTIC

테스트 코드를 이용해 `update()`로직을 진행했을때는 다음과 같은 쿼리가 나갔다.
> `@DynamicUpdate`를 붙혀두었다.

```kotlin
@SpringBootTest
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
internal class TestServiceTest{

    @Autowired
    lateinit var service:TestService

    @BeforeAll
    fun beforeAll(){
        service.save()
    }

    @Test
    @DisplayName("setActive Test")
    fun activeTest(){
        service.updateActive()
    }
}
```

```sql
select
    testentity0_.id as id1_0_,
    testentity0_.count as count2_0_,
    testentity0_.is_active as is_activ3_0_,
    testentity0_.version as version4_0_ 
from
    test_entity testentity0_ 
where
    testentity0_.id=?
---
update
    test_entity 
set
    is_active=?,
    version=? --- (1)
where
    id=? 
    and version=? --- (1)
---
select --- (2)
    version as version_ 
from
    test_entity 
where
    id = ?
```

2가지 재밌는 사항이 존재한다.

1. 서비스코드에는 version을 건드리는 것이 없었지만 이 값이 실제로 업데이트 되고 있다. (결과적으로 version이 1증가한다.)
2. 그리고 version에 대한 select문이 한번 더 나간다.

`2번 쿼리는 왜 나가는 걸까?` 를 생각해보면 단순하다. 만약 이 값이 초반과 달라졌다고 하면 `OptimisticLockingFailureException`이 발생한다. 
> 이제 어느정도 감이 잡히기 시작했다.! 트랜잭션의 처음과 끝에 버전을 2번 감시하는 것이다.

### LOCKTYPE - OPTIMISTIC_FORCE_INCREMENT

리포지토리의 LOCKTYPE만 변경하여 다시 해당 메서드를 호출해보면 다음과 같다.

```sql
select
    testentity0_.id as id1_0_,
    testentity0_.count as count2_0_,
    testentity0_.is_active as is_activ3_0_,
    testentity0_.version as version4_0_ 
from
    test_entity testentity0_ 
where
    testentity0_.id=?
---
update --- (1)
    test_entity 
set
    is_active=?,
    version=? 
where
    id=? 
    and version=?
---
update --- (2)
    test_entity 
set
    version=? 
where
    id=? 
    and version=?
```

달라진 차이점을 보자면 update문이 두번 나간것이다. 결론적으로 이야기하면 version이 1이 아닌 2가 올라가게 된다.

<br/>

그냥 `@Lock(LockModeType.OPTIMISTIC)` 의 경우 version이 1->2로 증가한 반면, `@Lock(LockModeType.OPTIMISTIC_FORCE_INCREMENT)`의 경우는 version이 1->3으로 증가했다.

---

### 2개 트랜잭션을 열어 실습하기

#### 위에서 두가지 방식의 공통, 차이점?

__가장 큰 차이로는__ 
후자의 방식을 보자면 전자의 마지막 쿼리 `select version as version_` 과 같은 버전확인을 트랜잭션 종료시에 진행하지 않고 version에 대한 업데이트 쿼리만 두번 나가고 있다. 

<br/>

또한 버전 관리 방식이 다르다. 아래와 같은 테스트 코드가 있다고 하자.

```kotlin

@Test
fun test_success(){
    val em1 = emf.createEntityManager()
    val tx1 = em1.transaction
    tx1.begin()
    val entity1 = em1.find(TestEntity::class.java, 1L, LockModeType.NONE)
    println("=== 초기버전: ${entity1.version}")
    tx1.commit()
    
    val one = service.getIdOne()
    println("=== 이후버전: ${one.version}")
}
```

엔티티를 가져오는 과정만 있을 뿐, 아무런 업데이트 사항이 없다. 전자와 후자를 사용할때 버전은 어떻게 달라질까?

- `LockTypeMode.OPTIMISTIC`

```text
Hibernate: 
    select
        testentity0_.id as id1_0_0_,
        testentity0_.count as count2_0_0_,
        testentity0_.is_active as is_activ3_0_0_,
        testentity0_.version as version4_0_0_ 
    from
        test_entity testentity0_ 
    where
        testentity0_.id=?
=== 초기버전: 1
Hibernate: 
    select
        testentity0_.id as id1_0_,
        testentity0_.count as count2_0_,
        testentity0_.is_active as is_activ3_0_,
        testentity0_.version as version4_0_ 
    from
        test_entity testentity0_ 
    where
        testentity0_.id=?
Hibernate: 
    select
        version as version_ 
    from
        test_entity 
    where
        id =?
=== 이후버전: 1
```

당연히 아무런 변화도 주지 않았으므로 버전에 변화가 없다.

- `LockTypeMode.OPTIMISTIC_FORCE_INCREMENT`

```text
Hibernate: 
    select
        testentity0_.id as id1_0_0_,
        testentity0_.count as count2_0_0_,
        testentity0_.is_active as is_activ3_0_0_,
        testentity0_.version as version4_0_0_ 
    from
        test_entity testentity0_ 
    where
        testentity0_.id=?
=== 초기버전: 1
Hibernate: 
    select
        testentity0_.id as id1_0_,
        testentity0_.count as count2_0_,
        testentity0_.is_active as is_activ3_0_,
        testentity0_.version as version4_0_ 
    from
        test_entity testentity0_ 
    where
        testentity0_.id=?
Hibernate: 
    update
        test_entity 
    set
        version=? 
    where
        id=? 
        and version=?
=== 이후버전: 2
```

?!?! 버전이 바뀌었다. 아무런 업데이트가 진행되지 않더라도 이 모드는 버전을 1증가시킨다.
> 실제 업데이트가 일어난 경우는 2증가시킨다.

이것이 의미하는 바가 굉장히 크다. 만약 `LockTypeMode.OPTIMISTIC_FORCE_INCREMENT` 모드를 걸어 객체를 조회한 경우는 다른 트랜잭션에게 제한을 걸어버리는 의미와 같아진다. 
[언제 OPTIMISTIC_FORCE_INCREMENT lock mode를 사용할까?](https://www.logicbig.com/tutorials/java-ee-tutorial/jpa/optimistic-lock-force-increment-use-case.html)를 살펴보면 다음과 같은 상황에서 사용된다고 한다.

```text
LockModeType.OPTIMISTIC_FORCE_INCREMENT can be used in the scenarios 
where we want to lock another entity while updating the current entity. 
The two entities can be or cannot be directly related, but 
they might be dependent on each other from business logic perspective.
```

<br/>

__공통점으로는__ 버전 넘버를 통해 트랜잭션 격리성을 관리한다는 것이다. 이름에 걸맞게 race condition이 발생하지 않을 것이라고 생각하는 방식의 locking이며 이를 잠근다는 의미보다는 충돌이 났을때 이를 감지하는 의미가 더 크다. 또한 DB레벨에서의 '락'이 아니므로 다른 트랜잭션이 들어오는 것이 가능했다!
> 어플리케이션이 쓰기 작업보다 읽기 작업이 많은 경우 적합한 locking strategy에 해당한다.


<br/>

그럼 이제 이 두가지 방식을 가지고 다른 예시를 보자. 마찬가지로 `Jpa + Hibernate`를 이용해 진행하겠다. 

#### 성공흐름

```kotlin

@Test
fun test_success(){
    val em1 = emf.createEntityManager()
    val em2 = emf.createEntityManager()

    val tx1 = em1.transaction
    val tx2 = em2.transaction

    tx1.begin()
    val entity1 = em1.find(TestEntity::class.java, 1L, LockModeType.NONE)
    println("=== 초기상태: ${entity1.isActive}")
    entity1.updateActive()
    tx1.commit()

    tx2.begin()
    val entity2 = em2.find(TestEntity::class.java, 1L)
    entity2.updateActive()
    tx2.commit()

    val finalEntity = service.getOne()
    println("=== 완료상태 :${finalEntity.isActive}")

}

```

트랜잭션 2개를 열어 실제값이 어떻게 바뀌는지 확인해보기 위함이다. 초기값은 `false`이고, 트랜잭션 2번 각각에서 이 값을 반대로 바꾸는 연산을 진행하므로 완료 상태값은 다시 `false`가 나와야한다.

```text

Hibernate: 
    select
        testentity0_.id as id1_0_0_,
        testentity0_.count as count2_0_0_,
        testentity0_.is_active as is_activ3_0_0_,
        testentity0_.version as version4_0_0_ 
    from
        test_entity testentity0_ 
    where
        testentity0_.id=?
=== 초기상태: false
Hibernate: 
    update
        test_entity 
    set
        is_active=?,
        version=? 
    where
        id=? 
        and version=?
Hibernate: 
    select
        testentity0_.id as id1_0_0_,
        testentity0_.count as count2_0_0_,
        testentity0_.is_active as is_activ3_0_0_,
        testentity0_.version as version4_0_0_ 
    from
        test_entity testentity0_ 
    where
        testentity0_.id=?
Hibernate: 
    update
        test_entity 
    set
        is_active=?,
        version=? 
    where
        id=? 
        and version=?
Hibernate: 
    select
        testentity0_.id as id1_0_,
        testentity0_.count as count2_0_,
        testentity0_.is_active as is_activ3_0_,
        testentity0_.version as version4_0_ 
    from
        test_entity testentity0_ 
    where
        testentity0_.id=?
Hibernate: 
    select
        version as version_ 
    from
        test_entity 
    where
        id =?
=== 완료상태 :false

```

위 로직은 Optimistic Locking을 걸어주지 않아도 당연히 성공한다.

#### 실패사례

```kotlin
@Test
fun test_failure(){
    val em1 = emf.createEntityManager()
    val em2 = emf.createEntityManager()

    val tx1 = em1.transaction
    val tx2 = em2.transaction

    tx1.begin()
    val entity1 = em1.find(TestEntity::class.java, 1L, LockModeType.NONE)
    println("=== 초기상태: ${entity1.isActive}")
    tx2.begin()
    val entity2 = em2.find(TestEntity::class.java, 1L)
    entity2.updateActive()
    tx2.commit()
    entity1.updateActive()
    tx1.commit()

    val finalEntity = service.getOne()
    println("=== 완료상태 :${finalEntity.isActive}")

}
```

위 테스트는 반드시 실패해야한다. 2가지 트랜잭션이 서로 꼬여 있다. tx1이 커밋되기전, tx2가 이미 변경을 하고 지나간 상태이다.

- 락킹적용 X

엔티티의 `@Version`이 달린 칼럼과, 리포지토리의 `@Lock`을 모두 제거한 상태에서는 어떻게 바뀔까?

<img src="/assets/img/jpa/1.JPG">

성공해버린다.. 완료 상태는 `false`가 나와야하지만 `true`가 당당히 찍힌 것으로 확인되었다.
> Locking이 이렇게 중요하구나...

- 락킹 적용(LockTypeMode.OPTIMISTIC)

그럼 첫번째 옵션인 `LockTypeMode.OPTIMISTIC`을 적용해보자. 또한 엔티티에 `@Version`어노테이션을 적용한 칼럼도 추가하자.

<br/>

이후 테스트를 진행하면

<img src="/assets/img/jpa/2.JPG">


에러가 잡힌다! 보이다 싶이 `OptimisticLockException`이 발생하며, 다른 트랜잭션에 의해 이 값이 변경되었다고 한다. 굿!

- 락킹 적용(LockTypeMode.Optimistic_FORCE_INCREMENT)

`LockTypeMode.OPTIMISTIC_FORCE_INCREMENT`를 적용해도 위와 같은 에러가 잡히며 테스트가 실패한다.

## 결론

두가지 방식 모두 결국엔 `Application에서 제공하는 Optimistic Locking`이라는 점이다. 그렇기에 트랜잭션 중간에 다른 트랜잭션이 개입하더라도 오류가 발생하지 않고, 이후에 버전 체크에서 오류를 뱉어내개 하는 것이다. 이 점은 다음 포스트에서 설명할 `Pessimistic Locking`과 큰 차이점에 해당한다. (실제 DB의 락 획득과는 무관하다.)

<br/>

초입 부분에 설명하였듯이 `LockTypeMode.OPTIMISITC`과 `LockTypeMode.OPTIMISTIC_FORCE_INCREMENT`모드는 각각 `READ`와 `WRITE`이라는 이름또한 가지고 있다. 이러한 SEMANTIC 으로부터 오는 차이점을 어느정도 이해하고 있는 것이 중요하다. 

# 출처

- [StackOverFlow](https://stackoverflow.com/questions/15293275/semantic-of-jpa-2-0-optimistic-force-increment)
