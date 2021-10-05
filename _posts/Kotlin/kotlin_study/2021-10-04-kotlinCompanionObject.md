---
title: 코틀린의 CompanionObject vs 자바 Static
date: 2021-10-04 13:00:00 +0900
categories: [Kotlin,kotlin_study]
tags: [kotlin,jvm,companionObject,static,java]
---

## Kotlin + Companion Object 
[출처](https://www.bsidesoft.com/8187) [출처2](https://kotlinlang.org/docs/object-declarations.html#using-anonymous-objects-as-return-and-value-types)

<br/>

코틀린에서의 Companion Object는 자바의 Static과 엄연히 다르다. 그렇다면 왜 기존의 Static을 쓰지 않고 코틀린에와서 이렇게 넘어오게 됬는지, 그 연유를 찾아 올라가면 알 수 있지 않을까?

### 자바의 Static & 코틀린의 Object

자바에서의 static이 붙은 변수나 메서드를 각각 클래스 변수, 클래스 메서드라고 부르고 있다. static이 붙은 멤버는 클래스가 메모리에 적재될 때 자동으로 함께 생성되므로 인스턴스 생성 없이 접근이 가능하다.
> 이외는 인스턴스 변수, 인스턴스 메서드라고 이야기한다.

<br/>
코틀린에서도 이와 비슷한 논리로 companion object를 사용한 경험이 있다. 아래의 TEST나 method는 외부에서 . 을 이용해 접근이 가능하다.

```kotlin
class Vitriol{
    companion object{
        val TEST = "test"
        fun method(i:Int) = i + 10
    }
}
fun main(args: Array<String>){
    println(Vitriol.TEST);    //test
    println(Vitriol.method(1));   //11
}
```

하지만 여기서 눈여겨볼 선언방식이 나타난다. 이는 Companion 'Object'라는 것이다. 코틀린은 싱글턴을 언어 수준에서 만들어 준다. class 키워드 대신 object키워드로 생성할 경우 이는 싱글턴 방식으로 관리가 된다.

```kotlin
object VitriolSingleton{
    val prop = "나는 VitriolSingleton의 속성이다."
    fun method() = "나는 VitriolSingleton의 메소드다."
}
fun main(args: Array<String>){
    println(VitriolSingleton.prop);    //나는 VitriolSingleton의 속성이다.
    println(VitriolSingletonm.method());   //나는 VitriolSingleton의 메소드다.
}
```

이는 특정 클래스나 인터페이스를 확장해 만들 수 있으며 위처럼 선언문이 아니라 표현식으로 생성할 수 있다.

```kotlin
open class A(x: Int) {
    public open val y: Int = x
}

interface B { /*...*/ }

val ab: A = object : A(1), B {
    override val y = 15
}
```

다시 돌아오면 companion object는 클래스 내부에 정의되는 object의 특수한 형태라는 것이다.

### Companion Object이 static과 다른 이유 (1)

이 코드를 다시 한번 확인해 보자.

```kotlin
class Vitriol{
    companion object{
        val TEST = "test"
        fun method(i:Int) = i + 10
    }
}
fun main(args: Array<String>){
    println(Vitriol.TEST);    //test
    println(Vitriol.method(1));   //11
}
```

main 함수에서 보면 Vitriol.TEST는 사실 `Vitriol.Companion.TEST`의 축약본이다. 실제 import되는 문구를 확인해보면 더욱 명확해진다. 즉, companion object는 static과 다르게 클래스가 메모리에 적재되면서 함께 생성되는 동반 객체이고 이는 `클래스.Companion`으로 접근이 가능하다.
> 이렇게 언어적으로 지원하는 축약 표현 때문에 companion object가 static과 혼동이 되는 것이다.

### Companion Object이 static과 다른 이유 (2)

Companion Object를 가지고 다음과 같이 코드를 작성할 수도 있다.

```kotlin
fun main(args: Array<String>){
   val comp1 = Vitriol.Companion 
    println(comp1.prop)
    println(comp1.method())
    val comp2 = Vitriol 
    println(comp2.prop)
    println(comp2.method())
}
```

자바의 Static이라면 사용하지 못할 코드들이다. 이것이 가능한 이유는 companion 'object'에 해당하기 때문이다. 객체이므로 변수에 할당 할 수 있고, 이의 멤버들에 접근이 가능한 것이다. 이에 추가로 축약 표현 덕분에 `val comp2 = Vitriol`과 같은 표현이 가능해진다.
> Static키워드만으로는 클래스 멤버를 companion object처럼 하나의 독립된 객체로 여길 수 없다.! 이것또한 큰 차이점에 해당한다.

### Companion Object이 static과 다른 이유 (3)

(2)와 상당히 연결되는 부분에 해당한다. 객체이므로 새로운 이름을 할당해 줄 수 있다.

```kotlin
class Vitriol{
    companion object MyCompanion{  // -- (1)
        val prop = "나는 Companion object의 속성이다."
        fun method() = "나는 Companion object의 메소드다."
    }
}
fun main(args: Array<String>) {
    println(Vitriol.MyCompanion.prop)
    println(Vitriol.MyCompanion.method())

    val comp1 = Vitriol.MyCompanion 
    println(comp1.prop)
    println(comp1.method())

    val comp2 = Vitriol 
    println(comp2.prop)
    println(comp2.method())

    val comp3 = Vitriol.Companion // 에러
    println(comp3.prop)
    println(comp3.method())
}
```

이름을 지어주었다 하더라도, 클래스이름만으로 접근할 수 있지만 기본 참조이름 (.Companion)으로는 접근할 수 없게된다. 이 역시 자바의 static으로는 불가능한 영역에 해당한다.

<br/>

+) 추가로 우리는 `val comp2 = Vitriol` 처럼 클래스이름으로 companion Object에 접근하는 것을 확인했다. 이렇게 되면 클래스 내에 2개이상의 companion Object가 올 수 있을까?
> 당연히 불가능하다. 클래스 명만으로 companion object객체를 참조할 수 있기때문에 이를 애초부터 막아두었다. 이름을 별도로 부여해도 마찬가지이다!

<br/>
인터페이스에서도 companion object를 정의해 줄 수 있다. 물론 이점은 jdk8로 넘어와서 인터페이스에서 static method를 정의할 수 있는 점과 비슷하다. 
> 인터페이스에서 정의된 static들은 모두 final에 해당하며 이들의 override는 이루어질 수 없다. 코틀린에서 이 점은 어떻게 될까.

```kotlin
open class Parent{
    companion object{
        val parentProp = "나는 부모값"
    }
    fun method0() = parentProp
}
class Child:Parent(){
    companion object{
        val childProp = "나는 자식값"
    }
    fun method1() = childProp
    fun method2() = parentProp
}
fun main(args: Array<String>) {
    val child = Child()
    println(child.method0()) //나는 부모값
    println(child.method1()) //나는 자식값
    println(child.method2()) //나는 부모값
}
```

위 처럼 부모 / 자식의 companion object가 다른 이름이라면 자식이 부모의 companion object 멤버를 직접 참조할 수 있게 된다. 
> 사실 이점은 상속이라는 개념보다는 '참조'가 더 어울린다. 이미 컴파일 타임에 생성된 것을 super나 Parent.~ 로 직접 조회하는 것과 다를 것이 없다. 

만약 서로의 companion object가 같은 prop을 갖는다면 어떻게 될까 

```kotlin
open class Parent{
    companion object{
        val prop = "나는 부모"
    }
    fun method0() = prop
}
class Child:Parent(){
    companion object ChildCompanion{ 
        val prop = "나는 자식"
    }
    fun method1() = prop
    fun method2() = ChildCompanion.prop
    fun method3() = Companion.prop
}
fun main(args: Array<String>) {
    val child = Child()
    println(child.method0()) //나는 부모
    println(child.method1()) //나는 자식
    println(child.method2()) //나는 자식
    println(child.method3()) //나는 부모 (!)
}
```

실제 main부분의 method1을 보면 자식클래스의 companion object가 되어있다. 이것은 자바에서와 마찬가지로 '상속하여 override' 된것이 아니라, shadowing한 것에 해당한다. 

<br/>
추가로 ! 부분을 보면 기존의 Companion은 자식의 것이 아니다. (ChildCompanion이라는 이름을 붙혔기 때문에)여기서 Companion은 부모가 되므로 부모의 prop을 가져오게 된다.
> 만약 부모의 companion object에도 이름을 붙히게 되면 method3은 컴파일 에러를 뱉는다.

> 결론을 이야기하자면, 부모 자식간의 companion object에 정의된 멤버는 자식입장에서 접근할 수 있지만, 같은 이름을 사용하게 되면 shadowing되어 자식의 값으로 감추어짐을 알 수 있었다.

<br/>

마지막으로 companion object와 다형성을 엮어 볼 수 있다.

```kotlin
open class Parent{
    companion object{
        val prop = "나는 부모"
    }
    open fun method() = prop 
}
class Child:Parent(){
    companion object{
        val prop = "나는 자식"
    }
    override fun method() = prop
}
fun main(args: Array<String>) {
    println(Parent().method()) //나는 부모
    println(Child().method()) //나는 자식
    val p:Parent = Child()
    println(p.method()) //나는 자식
}
```
Child의 method()에서는 shadowing이 되어 '나는 자식'이라는 문구를 내놓게 된다. 마지막으로 p.method()의 경우에도 다형성에 의해 '나는 자식'이라는 문구를 던지게 된다.