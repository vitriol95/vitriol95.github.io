---
title: 코틀린 지네릭스 - 1
date: 2021-10-02 12:00:00 +0900
categories: [Kotlin,kotlin_study]
tags: [kotlin,jvm,generics]
---

## Variant in Java

#### 가장 먼저 다음의 상황을 가정해 보자.
- 1 - Animal이라는 클래스를 상속한 Dog이라는 클래스가 존재한다.
- 2 - Group<T> 라는 지네릭 인터페이스가 존재하며, 다음과 같은 추상 메서드가 존재한다
    - void insert(T item)
    - T fetch()

#### 이를 사용함에 있어 다음과 같은 문제를 맞닥뜨렸을 것이다.

- 3 - 하지만 `Group<Dog>`은 `Group<Animal>`의 하위 클래스가 아니다.

#### 2번에 대한 문제를 해결한다는 것은 곧 '형식 매개변수가 클래스 계층에도 영향을 주는 것'에 해당하며 '가변성(variance)을 준다'를 의미한다. <br>
> 그리고 우리는 지네릭스의 upper, lower bound를 클래스 선언부나 메서드 args에 붙혀 이 문제를 해결했을 것이다.
- `Group<T extends Animal>` or `Group<T super Dog>`
- `void insert(Group<T extends Animal> animalGroups)`

#### 코틀린에서도 당연히 위와 같은 지네릭스를 다룰 수 있는 키워드가 존재한다. 앞선 상황을 포함하여, 이들을 어떻게 해결하는지 알아보자.

## Variance in detail

#### 가장 먼저 다음과 같은 용어들 정리가 필요하다. 앞선 Animal과 Dog, Group의 예시를 그대로 가져오겠다.

1. 기본상황                 `Animal <- Dog `(Super - Sub relation)
2. 무변성(Invariance)     `Group<Animal> Group<Dog>` (No relation)
3. 공변성(Covariance)       `Group<Animal> <- Group<Dog>` 
> 코틀린에서는 out이라는 키워드가 사용 된다.
4. 반공변성(Contravariance) `Group<Dog> <- Group<Animal>`
> in 이라는 키워드가 사용된다.

#### 1번은 상속에 의해 이루어지는 당연한 관계에 해당하고, 2번은 일반 지네릭의 성질에 해당하므로 우리가 알아야 할 것은 3,4번에 해당한다.
> 코틀린에서는 형식 매개변수에 in, out등의 키워드가 존재하지 않으면 모두 2의 관계를 가지게 된다.

## 공변(Covariance) - out 키워드

#### 3번의 상황으로 매개변수의 상하관계가 인스턴스 자료형 관계로도 이어지는 것을 이야기 한다. 이때 out키워드를 사용한다.
> 앞선 Animal, Dog, Group으로 예시를 들어 보자.

~~~kotlin
    class Group<out T>(private val elem:T){
        fun getT(): T = elem
    }
    open class Animal{
        open fun sounds(){
            println("animal's sound")
        }
    }

    class Dog: Animal(){
        override fun sounds(){
            println("dog barks")
        }
    }

    fun main(){
        val animalGroup : Group<Animal> = Group<Dog>(Dog())
        val anyGroup : Group<Any> = Group<Animal>(Animal())
        animalGroup.getT().sounds() // "dog barks"
    }
~~~
#### 이처럼 `Group<Animal>`의 자료형에 `Group<Dog>`의 인스턴스를 부여할 수 있게 되고, 우리는 앞선 문제점을 해결할 수 있게 된다.

> `class Group<out T:Animal>` 로 UpperBound를 줄 수 있다! 이렇게 되면 `Group<Any>`로 캐스팅은 불가능해진다.

<br> 

#### 그럼 이런 성질을 활용하는 예시또한 살펴보자. (함수의 param으로 사용)

~~~kotlin
fun copy(from: Array<Any>, to: Array<Any>){
    assert(from.size == to.size)
    for (i in from.indices)
        to[i] = from[i]
}

fun main(){
    val ints: Array<Int> = arrayOf(1,2,3)
    val any = Array<Any>(3){""}
    copy(ints, any)
}
~~~
어떤 작업을 하려 하는지는 알 것이다. 위 코드를 실행하면 당연히 컴파일 오류가 발생한다. `Array<Any>`와 `Array<Int>`는 Invariance 관계이기 때문이다. 위 코드가 작동되게 하려면 copy 함수를 다음과 같이 수정하면 된다.

~~~kotlin
fun copy(from: Array<out Any> to: Array<Any>){
    assert(from.size == to.size)
    for (i in from.indices)
        to[i] = from[i]
}
~~~
> out키워드가 들어간 순간 Any와 상속관계가 있는 클래스들이 Array의 Type param으로 들어올 수 있다.

> 여기서도 마찬가지로, 자바의 `Array<? extends Object>`와 비슷한 면이 있다.

#### 위 작업에서, 파라미터 from의 성질은 다음과 같다.
- Simple Array가 아닌, Restricted Array가 된다. 
    - 즉, 위의 예시에서는 `Array<Int>`로 type projection이 진행된 형태이다.
- 따라서 우리는 type parameter T를 리턴값으로 얻어올 수 있다.

#### 이렇게 우리는 projection된 type parameter를 return할 수 있기 때문에 이를 out 키워드를 이용해 사용하게 되는 것이다.
> can only be produces and never consumed
<br>

## 반공변성(Contravariance) - In

#### out과 반대의 방향으로 관계를 맺게 해준다. 이번엔 바로 예시로 부터 접근해보자.

~~~kotlin
interface Comparable<in T> {
    operator fun compareTo(other: T): Int
}

fun demo(x: Comparable<Number>){
    x.compareTo(1.0)
    val y: Comparable<Double> = x 
}
~~~
위 코드는 실제 Comparable 인터페이스의 코드에 해당한다. in의 경우 out과 비교했을때, 관계가 역전된 상태이므로
`Comparable<Double>`타입이 `Comparable<Number>`인스턴스를 가져갈 수 있다. 따라서 Contra- 라는 접두어가 붙는 것이다.

#### 다른 예시를 하나 더 살펴보자 (함수의 param)
~~~kotlin
fun fill(dest: Array<in String>, value: String){
    dest[0] = value
}
~~~
여기서 dest에 String에 상위 타입에 해당하는 CharSequence의 array등이 들어갈 수 있게되는 것이다.
> 마찬가지로 자바의 Array<? super String>과 비슷한 면이 있다.

## 궁금증

#### 위의 in, out과 같은 키워드를 사용할 때 특이한점을 느꼈다. 
- out T가 붙은경우(covar) 리턴값으로 T가 나올 수 있으나, input값으로 쓰이지 못한다(컴파일 에러)
- in T가 붙은경우(contravar) input 값으로 T가 쓰일 수 있으나, 리턴값으로 쓰이지 못한다(컴파일 에러)

#### 하지만 이 특이한점들은 앞선 co,contra - var 스키마(근본적으로는 super-sub class)의 이해에 연장선에 해당했다.
기본적인 2원칙을 상기해보자.
- ㄱ. A subtype must accept ___at least the same range of types___ as its supertype declares.
- ㄴ. A subtype must return ___at most the same range of types___ as its supertype declares.

1. 가장 먼저 out이 붙은 경우 T의 하위 클래스들이 타입 파라미터로 자리할 수 있다.
- 우리는 리턴값으로 projected parameter인 T를 뱉을 수 있다.
- 하지만 T를 함수의 input으로 사용하여 다룰 수 없다. `Group<out Animal>`로 예시를 들어보자 
    - `insert(newAnimal: T)` 로 선언한 순간 이 T엔 projected parameter가 자리한다.
    - 이때 `T.method()` 를 호출할 수 없다. T는 Animal의 하위타입이기에 없는 프로퍼티들이 존재할 수 있기 때문이다.
    - 즉 2원칙중 ㄱ에 위배된다.

2. in이 붙은경우 T의 상위 클래스들이 타입 파라미터로 자리할 수 있다.
- T를 함수의 input으로 받아 다룰 수 있게된다.
    - `Group<in Dog>`의 경우 Dog은 상위 클래스들의 프로퍼티를 모두 가지고(or 오버라이딩) 있기때문에 `T.method()`, `T.property()`에 접근이 가능하다.
- 하지만 리턴값으로 projected parameter인 T를 뱉을 수 없다.
    - 앞서 이야기한 2원칙의 ㄴ에 위배되므로 지네릭이 깨지게 된다. 

## 참고자료
- [https://kotlinlang.org/docs/generics.html](https://kotlinlang.org/docs/generics.html)
- [https://typealias.com/guides/ins-and-outs-of-generic-variance/](https://typealias.com/guides/ins-and-outs-of-generic-variance/)
