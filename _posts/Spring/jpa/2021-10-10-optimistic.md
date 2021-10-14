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

## 적용

JPA에서 낙관적 락을 사용하기 위해서는 `@Entity`내부에 `@Version` 어노테이션이 붙은 변수를 구현해 줌으로써 가능하다. 이때 다음과 같은 주의사항이 존재한다.

- 각 엔티티 클래스에는 `@Version`속성이 하나만 있어야 한다.
- 여러 테이블에 매핑된 엔티티의 경우, 기본 테이블에 존재해야한다.
- 타입은 int, Integer, long, Long, short, Short가 되며, Timestamp도 가능하다.
> 코틀린의 경우 맘편하게 int로 설정하면 될것이다.

JPA는 select가 일어날 때, 버전 속성값을 가지고 있다가 트랜잭션이 업데이트를 치기 전에 버전 속성을 다시 확인한다. 이 과정에서 버전 정보가 변경된다면 `OptimisticLockException`이 발생하게 된다.

## 실습해보기 - 준비

코틀린 1.5.31, Jdk 11버전으로 진행할 생각이다. 의존성은 data jpa, h2인메모리 db를 사용하며 테스트코드는 junit5를 이용해 작성하겠다.

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