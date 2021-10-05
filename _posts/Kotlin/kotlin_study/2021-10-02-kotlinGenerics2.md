---
title: 코틀린 지네릭스 - 2
date: 2021-10-02 13:00:00 +0900
categories: [Kotlin,kotlin_study]
tags: [kotlin,jvm,generics]
---
## In Out

지난 내용을 요약하면 in / out으로 지네릭에 flexibility를 주는 것을 보았다. projected된 지네릭 파라미터를 읽고싶을 때에는 out을 사용하고, 이를 내부에서 사용하고 싶을때에는 in이라는 키워드를 사용하면 된다.

#### 하지만 이런 궁금증도 든다, 이 둘을 동시에 사용하는 방법이 있을까??

가장 먼저 생각할 수 있는 방법으로는 projected parameter를 Read, Write를 할 수 있는 각각의 out, in 키워드가 붙은 지네릭 클래스를 만들고 이둘을 상속하는 것이 있다.

<br>
하지만 좀 더 일반적이면서, 권장하는 방법을 코틀린에서는 제공한다. 전에 이야기 한 두가지 원칙을 중심으로 보자

- ㄱ. A subtype must accept at least the same range of types as its supertype declares.
- ㄴ. A subtype must return at most the same range of types as its supertype declares.

## SuperGroup

#### 앞서 이야기한 `Group<T>`가 모든 projected parameter를 받고, 리턴하기 위해서는 앞선 두 원칙을 지켜야한다. 
> 이를 SuperGroup이라고 하겠다.

그렇다면, SuperGroup의 모양은 이렇게 생겨야 할 것이다.
~~~kotlin
interface SuperGroup {
    fun insert(item: Nothing) // Nothing은 모든 클래스의 하위타입에 해당하므로 확장이 가능하다.
    fun fetch(): Any? // Any?의 경우 모든 클래스의 상위 타입에 해당하므로 subtype으로 좁히는 것이 가능하다.
}
~~~
> 이 인터페이스를 Group이 상속해서 사용해도 괜찮을까? 
안타깝게도 되지 않는다.. 코틀린에서는 지네릭이 아닌 normal class / interface의 상속에서 contravariant argument types를 지원하지 않는다.

#### 마치 다음과 같은 상황이다.
- 전제조건
    - A가 B의 상위 클래스
    - Super가 Sub의 상위 클래스

~~~kotlin
open class Super {
    open fun execute(arg: B) {
        // ...
    }
}

class Sub : Super() {
    override fun execute(arg: A) {
        // ...
    }
}
~~~
super에서 sub으로 내려가면서, override fun의 argument는 super로 가고있기에 다음과 같은 상황이 contravariant argument에 해당한다.
> 이를 금지해둔 이유는 오버로딩과의 혼선을 피하기 위해서이다. 따라서, Sub 클래스의 execute fun에서 override 키워드를 지우면 컴파일 에러도 뱉지 않는다.

#### 그렇다면 어떻게 해야하는가? --> 일반 클래스/인터페이스는 불가하므로 지네릭을 써야한다.

동시에, in과 out 스키마를 합치시켜보면 두가지 옵션이 나올 수 있다.
- In-projections set the return types to Any?
- Out-projections set the argument types to Nothing

이를 도식화 해 둔것이 아래의 그림에 해당한다. 

<img src="/assets/img/kotlin_study/1.JPG">

즉, `Group<in Nothing>` 혹은 `Group<out Any?>`으로 사용할 경우 우리가 원하는 목표를 이룰 수 있다.

## Star-Projections

#### 하지만 코틀린에서는 더 나은 관용적인 방법을 제시한다. 이것이 star projections이다. 기존 방법들을 살펴보자.

1. `Group<in Nothing>`

```kotlin
interface Group<T : Animal> {
  fun insert(member: T): Unit
  fun fetch(): T
}

fun readIn(group: Group<in Nothing>) {
  val item = group.fetch()
}
```

readIn 함수를 통해 얻은 item은 어떠한 타입을 지니게 될까. 우리는 T: Animal 이라는 상한선을 주었기 때문에, 최소 Animal보다 제너럴한 타입이 올 것이라 예측할 수 있다. 따라서 코틀린이 이 item을 Animal이라고 예측해주기를 바란다.

#### 하지만 실제로는 그렇지 않다, 코틀린은 item의 타입을 여전히 Any?로 유지하게 된다.

2. `Group<out Animal>` (상한선을 Any?에서 Animal로 제한)

```kotlin
fun readOut(group: Group<out Animal>) {
  val item = group.fetch()
}
```

out projection은 어떻게 될까. 이 경우에는 item의 타입을 정확히 Animal로 가져다 주게 된다. 즉, item.walk()등의 Animal의 함수들을 사용할 수 있게 되는 것이다.

3. `Group<*>` StarProjections

```kotlin
fun readStar(group: Group<*>) {
  val item = group.fetch()
}
```
star projections를 사용한 경우, 앞선 2와 같이 item의 타입은 Animal로 가져다 주게 된다. 아직까진 2와 3방법에서 큰 차이는 없어보인다.

#### 만약, `Group<T>`의 제한을 좀 더 줄여보면 어떻게 될까?? (:Animal -> :Dog)

```kotlin
interface Group<T : Dog> {
  fun insert(member: T): Unit
  fun fetch(): T
}
```

1. 1번의 경우에는 그대로 item의 타입을 Any?로 유지하게 된다.
2. 2번의 경우 컴파일 에러를 뱉는다. Dog이 Animal보다 하위타입에 해당하기 때문이다.
    - 따라서 우리는 readOut의 arg를 `Group<out Dog>`으로 수정해주어야 한다.
3. 3번의 경우 컴파일이 진행되며 item의 타입은 Dog으로 매칭된다. 👍
> 이제 왜 얘를 써야하는지 알거같다.

## `<*>`는 `<Any?>`와 어떻게 다른가.

#### `Array<Any?>`와 `Array<*>`의 예시를 보자

```kotlin
fun acceptAnyArray(array: Array<Any?>) {}
fun acceptStarArray(array: Array<*>) {}

val arrayOfStrings = arrayOf("Hello", "Kotlin", "World")

acceptAnyArray(arrayOfStrings)  // Compiler error = Required: Array<Any?> Found: Array<String>
acceptStarArray(arrayOfStrings)
```

어찌 보면 당연한 결과이다. Array<Any?>는 Array<String>의 supertype이 아니기 때문이다. 
#### 하지만 star-projections시에는 컴파일이 된다. 앞서 살펴 보았듯이, 지네릭의 동작원리를 가져오기 때문이다.

<br>

#### 재미있는 실험이 하나 더 있는데, Array가 아니라 List로 실험할 경우 두가지 모두 제대로 컴파일이 된다.

```kotlin
fun acceptAnyList(list: List<Any?>) {}
fun acceptStarList(list: List<*>) {}
val listOfStrings: List<String> = listOf("Hello", "Kotlin", "World")
acceptAnyList(listOfStrings)
acceptStarList(listOfStrings)
```

왜 List는 되는 것일까? --> 지네릭이 숨어있다는 것이다. 즉, Standard Library인 Array의 정의와 달리 List의 경우에는 declaration-site에서 지네릭이 정의가 되어있다. 이는 실제 코드를 까보면 어렵지 않게 찾을 수 있다(이는 자바에서 array가 공변타입인것과 같다).


## 참고자료
- [https://typealias.com/guides/star-projections-and-how-they-work/](https://typealias.com/guides/star-projections-and-how-they-work/)
- [https://kotlinlang.org/docs/generics.html#star-projections](https://kotlinlang.org/docs/generics.html#star-projections)
- [https://typealias.com/guides/ins-and-outs-of-generic-variance/](https://typealias.com/guides/ins-and-outs-of-generic-variance/)
