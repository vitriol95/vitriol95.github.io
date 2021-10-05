---
title: 코틀린 var, val 그리고 inline
date: 2021-10-02 11:00:00 +0900
categories: [Kotlin,kotlin_study]
tags: [kotlin,jvm,var,val]
---

## 인터페이스 상에서의 var과 val의 차이

코틀린에서 'var속성과 val속성을 사용한다.' 라는 것은 결국 '속성 처럼' 보이는 문법을 쓰게 해주는 getter나 setter를 만들어주는 것이다. 또한 이러한 특징은 추상 클래스나 인터페이스에서 추상 수준의 var이나 val을 사용할 수 있게 해준다.

<br/>
그렇다면 다음과 같은 선언을 보자. 이번 프로젝트에서 프로젝션을 사용할 때 자주 썼던 선언방식에 해당한다.

```kotlin
interface Test{
  val a:Int
  var b:String
}
```
이는 자바로 compile해보면 다음과 같다. val은 getter를 만들어내는 것이고, var의 경우는 setter까지 만들어 준다.

```java
interface Test{
  int getA();
  String getB();
  void setB(v:String);
}
```

### 그렇다면 상속에서는

자바에서는 `1. 메서드 명이 일치해야 하고`, `2. 각 인자는 완전히 동일한 형을 가지고 있어야한다.` 조건들을 만족해야 overriding을 진행할 수 있다. 하지만, jdk1.5 버전 이후 override한 메소드의 반환형이 원래 메서드 반환형의 자식 계열이면 허용하게 된다. 아래와 같은 코드를 성립하게 해주는 것이다.

```java
interface Test{
  Object getB()
}
 
class Concreate implements Test{
  @override
  String getB(){return "abc";}
}
```

이는 어찌보면 당연한 객체지향의 원리에 해당한다. 외부에서는 Concrete의 get메서드에 접근할 때, Test의 메서드로 접근하게 되는데, 결국 String을 반환해도 공변(covariance)에 의해 Object형을 반환했다는 점은 문제가 없다.

> 이 내용은 마찬가지로 코틀린에서도 해당한다.

그렇다면 val, var을 변성 관점에서 한번 따져보자.

```kotlin
interface Num{
  val num:Number
}

inline class NumInt(override val num:Int):Num
inline class NumFloat(override val num:Float):Num
inline class NumDouble(override val num:Double):Num
```

이렇게 구상형을 전개할 수 있고, 이 값은 문제가 되지 않는다. 만약 NumInt를 자바코드로 본다면 다음과 같다.

```java
class NumInt implements Num{
  private int num;
  NumInt(v:int){
    num = v;
  }
  @override
  int num(){return num;}
}
```
결국 val 타입을 사용하게 되면 손쉽게 구현 클래스에서 상속관계에 있는 하위형으로 바꾸어 쓸 수 있다는 것이다. 또한 이 값은 노출할 때 가리키는 형에 따라 다른 형으로 인식이 된다.

```kotlin
val numInt = NumInt(3)
numInt.num // Int
 
val num:Num = numInt
num.num // Number
```

> 추상형으로 인식하면 Num수준에서 바라보기 때문에 Number형이 된다. 따라서 추상 클래스에서 이 속성을 사용하려 할 때, 구현형이 지정한 num의 형식을 인식하는 것은 불가능하다.

이렇게 자유롭게 변경될 수 있는 이유로는 getter메소드라서 반환형에 대한 공변지원 때문에 가능하다. ___하지만 var는 다르다___
var는 setter에서 인자로 해당 형을 받기 때문에 구현형에서 무공변때문에 그대로 추상형을 사용해야 한다. 즉 다음과 같은 코드는 컴파일 에러를 뱉는다.

```kotlin
interface Num{
  var num:Number
}
 
class NumInt(override var num:Int):Num //error
```

이전에 배웠었던 변성의 개념으로 들어가서 보자면 훨씬 명확하다.

```kotlin
interface Num{
  fun getNum():Number
  fun setNum(v:Number)
}
 
class NumInt:Num{
  override fun getNum():Int{}
  override fun setNum(v:Int){}
}
```
var는 setter의 인자로 Number를 확정하고 있기에, 상속한 NumInt에서 v에 int형을 지정하는 것은 overriding 조건에 위배되는 것이다.

<br />
그렇다면 이 문제에 대한 해결책은 무엇일까? 대안은, 추상형에서는 val로 지정하고 구현형은 var로 override하는 방법이다.

```kotlin
interface Num{
  val num:Number
}
 
class NumInt(override var num:Int):Num
```

> 추상형에서는 getter만 내려주고, 구상층에서는 이를 var로 선언하여 setter도 만질 수 있게 해주는 것이다.

즉, 아래와 같은 코드가 만들어지는 것이다.

```kotlin
interface Num{
  fun getNum():Number
}
 
class NumInt:Num{
  override fun getNum():Int{}
  fun setNum(v:Int){}
}
```
getter는 오버라이드 한것으로 처리를 하는 것이고, setter는 새로 생성된 함수로 사용하는 것이다. 

#### [출처](https://www.bsidesoft.com/8201)

## Inline class

이전에 inline 함수에 대한 정리는 했는데, 이번엔 inline class도 함께 정리해보려한다. inline function은 일반 함수와는 다르게 실제 사용된 위치에 호출되어 'inline'으로 복사되어 진행된다. 일반 함수가 객체를 생성해 진행하는 것과 비교하면 런타임시에 오버헤드가 작다고 이야기 할 수 있다.
> 물론 구현부분이 길면 안되겠다.

### 언제 inline class를 사용하는지

만약 우리가 하나의 타입을 만들어서, 그 타입만 가질 수 있는 여러가지 동작들을 정의하고 싶을때가 있다. 예를 들어 Unix timestamp는 기본적으로 Long에 해당하지만, 이를 캘린더 타입으로 변환하는등의 작업을 해주고 싶다고 하자.
> 일반 클래스로도 되지않나? 라고 생각이 들긴하다. 아래에서 여러 방법들을 살펴보자.

1. type alias

typealias 키워드를 타입의 별칭으로 삼아 직관적으로 사용하는 방법이 존재한다. 예를 들어, 아래와 같은 코드로 작성해 볼 수 있다.

```kotlin
typealias UnixMillis = Long

fun UnixMillis.toCalendar(): Calendar {
    return Calendar.getInstance().also {
        it.timeInMillis = this
    }
}

fun main() {
    val unix: UnixMillis = System.currentTimeMillis()
    val calendar: Calendar = unix.toCalendar()
}
```

이후 확장함수로 toCalender를 정의하면 UnixMills타입이 Calendar로 변환될 수 있음을 알 수 있다. 하지만, 위 방식은 단점이 너무 많다.

- UnixMills의 타입은 Long에 지나지 않는다. 따라서 (1L).toCalendar()도 사용될 수 있다. 이는 자바로 치면 `long unix = System.currentTimeMillis();` 과 다름없으며, 이는 많은 버그를 불러올 수 있다.
> 예를 들어, 화씨와 섭씨는 같은 double타입에 해당되더라도 쓸 수 있는 상황이 다르며, 하나의 핸들러로 처리된다면 큰 오류를 불러올 수 있는 것과 같다.

2. Wrapper Class

Wrapper 클래스를 만드는 것으로 이 문제를 피해볼 수 있다. 이 방법은 가장 흔히 생각할 수 있는 방법이며, 나 또한 이런식으로 진행한적이 있었다.

```kotlin
class UnixMillis(private val millis: Long) {
    fun toCalendar(): Calendar {
        return Calendar.getInstance().also {
            it.timeInMillis = millis
        }
    }
}
```

이렇게 되면 더이상 Long타입에서 toCalendar를 호출하는 것은 불가능하다. 여기까지는 아주 좋다. 만약 이 코드가 자바화 되었다면 어떻게 보여질까?

```java
public static final void main() {
   UnixMillis unix = new UnixMillis(System.currentTimeMillis());
   Calendar calendar = unix.toCalendar();
}
```

어? 뭔가 예상했던 바람직한 방법과는 다르다 new UnixMills() 라는 생성자를 이용해서 계속 객체를 생성해 사용하기 때문에 이 방법마저도 완전히 최적화된 방법이라고 할 수 없다. 

3. 인라인 클래스

마지막으로 앞서 구현했던 wrapper 클래스의 value이라는 키워드를 붙혀보자.
> inline이라는 modifier는 deprecated되었으며, @JvmInline 어노테이션을 붙혀주어야 JVM이 이를 인식하게 된다.


```kotlin
@JvmInline
inline class UnixMillis(private val millis: Long) {
    fun toCalendar(): Calendar {
        return Calendar.getInstance().also {
            it.timeInMillis = millis
        }
    }
}
```

그리고 이 코드의 디컴파일된 자바 코드를 보면?


```java
public static final void main() {
   long unix = UnixMillis.constructor-impl(System.currentTimeMillis());
   Calendar calendar = UnixMillis.toCalendar-impl(unix);
}
```

실제 UnixMills라는 자료형은 쓰이지 않고, 기존 Wrapper 클래스에서 사용되던 함수들이 static 함수로 정의된 헬퍼 클래스로 변했다. 여기서는 constructor-impl, toCalender-impl이라는 유틸리티 함수가 사용되었고 long자료형이 그대로 사용되고 있었다. 이처럼 인라인클래스를 사용하면 코드를 최적화 해주고 안전한 사용법을 강제해 줄 수 있는 것이다.

### Inline Class의 특징 1

인라인 클래스가 감싸고 있는 값은 주 생성자로 전달할 수 있고, 이 때문에 인라인 클래스는 주 생성자로 하나의 인자만 가져야한다. 또한 var이나 val을 붙여 속성으로 만들어 주어야 한다.
> 앞선 UnixMills의 경우 Long타입의 millis라는 속성만 가지고 있어야 했다.

<br/>
다음과 같은 코드를 보며 특징들 또한 확인할 수 있다.

```kotlin
@JvmInline
value class Name(val s: String) {
    init {
        require(s.length > 0) { }
    }

    val length: Int
        get() = s.length

    fun greet() {
        println("Hello, $s")
    }
}

fun main() {
    val name = Name("Kotlin")
    name.greet() 
    println(name.length)
}
```

일반적인 클래스와 마찬가지로 프로퍼티와 함수, init블록을 지정해 줄 수 있다. main에서 호출한 greet()과 length는 모두 실제로static method로 호출이 된다. 또한 inline 클래스의 프로퍼티는 backingfield를 가질 수 없다. 오직 length와 같은 simple computable properties만 가질 수 있으며 lateinit과 같은 키워드를 사용할 수 없다. (static으로 활동하게 되므로)

### Inline Class의 특징 2

아래와 같이 구현 클래스의 역할도 가능하다. 하지만 이렇게 구현한 inline클래스는 반드시 final이된다.

```kotlin
interface Printable {
    fun prettyPrint(): String
}

@JvmInline
value class Name(val s: String) : Printable {
    override fun prettyPrint(): String = "Let's $s!"
}

fun main() {
    val name = Name("Kotlin")
    println(name.prettyPrint()) // 역시 static method로 호출된다.
}
```

Mangling이라는 특징또한 존재한다. 인라인클래스는 그들의 underlying type으로 컴파일이 되기에, 다음과 같이 이름이 겹치는 충돌 상황에서 문제가 생길 수 있다.

```kotlin
@JvmInline
value class UInt(val x: Int)

// Represented as 'public final void compute(int x)' on the JVM
fun compute(x: Int) { }

// Also represented as 'public final void compute(int x)' on the JVM!
fun compute(x: UInt) { }
```
이를 방지하기 위해서는 function의 이름에 stable한 hashcode를 넣어주는 방식을 채용할 수 있다. 즉, `fun compute(x: UInt)`라는 function이 `public final void compute-<hashcode>(int x)`처럼 컴파일링되게 해주어야 하는 것이다.

> 혹은 다음과 같은 방식으로 피해갈 수도있다. 이는 자바코드에서 사용될때 유용하다.
```kotlin
@JvmName("computeUInt")
fun compute(x: UInt) { }
```


#### [출처](https://kotlinlang.org/docs/inline-classes.html#inheritance)