---
title: 코틀린의 extension, inline 함수
date: 2021-10-04 14:00:00 +0900
categories: [Kotlin,kotlin_study]
tags: [kotlin,jvm,java]
---

## Extension Function

코틀린 내에 어떤 클래스의 메서드를 정의하는 데에는 두가지 방법이 있다. 하나는 확장함수, 다른 하나는 멤버 변수로써의 함수에 해당한다.

```kotlin
class Foo { ... }

fun Foo.bar() {
    // Some stuff
}
```

```kotlin
class Foo {

    ...

    fun bar() {
        // Some stuff
    }
}
```

그렇다면 이러한 extension function은 어떨때 쓰이는건지 궁금했다.

<br/>

#### 존재하는 API의 확장이나, 추가를 하고싶을 때 사용하면 좋다. 

이는 클래스에 새로운 기능을 부여하는 idiomatic한 방법에 해당한다. 예를 들 수 있는 것은 Jackson-Kotlin 모듈에해당한다. [출처](https://github.com/FasterXML/jackson-module-kotlin/blob/master/src/main/kotlin/com/fasterxml/jackson/module/kotlin/Extensions.kt#L15-L32)
이 모듈을 살펴보면 기존 자바에서 사용하던 ObjectMapper라는 클래스에 기능을 확장하여 사용하고 있는데, 코드를 살펴보면 `TypeReference` 와 generic을 이용해 확장하고 있었다.

> 즉, 코드가 original하게 자바로 쓰여있고 이 클래스에 새로운 메서드나 프로퍼티를 제공하고자 할 때 사용한다. 당연히 기존의 코드는 변경할 수 없고 special한 function의 추가만 가능하다.

<br/>
그럼 자바에서는 이 extension Function을 어떻게 불러올까? 다음과 같은 extension Function이 있다고 해보자.

```kotlin
Strings.kt
fun String.isEmail() : Boolean {
   // ~~
}
```

이는 자바로 다음과 같이 선언이 된다.

```kotlin
class StringUtils {
    public static boolean isEmail(String email) {
        // ~~
    }
}
```
당연히 사용하는 부분에서도 차이가 존재한다. 기존 코틀린에서는 `somethingString.isEmail()`과 같이 인스턴스 메서드로 사용할 수 있지만 자바에서는 `StringsKt.isEmail("example@example.com")` 과 같이 caller를 첫번째 argument로 넘겨주어야 한다. 코틀린 공식 docu에 따르면 다음과 같이 정의 되어있다.

> Extensions do not actually modify classes they extend. By defining an extension, you do not insert new members into a class, but merely make new functions callable with the dot-notation on variables of this type.

#### Null Safety를 추가하고 싶을때.

예를 들어, String의 extension function으로 보면 다음과 같다.
`String?.isNullOrBlank()` 라고 정의한 확장함수는 null이라고 하더라도 메서드를 실행시킬 수 있게 해준다. (널 체크를 하지 않더라도) 이러한 유연함을 제공한다는 측면에서 굉장히 좋은 것 같다. 이외에 의무적으로 써야하는 상황도 존재한다.

#### mandatory한 상황 - inline default func in interface

만약, interface에서 inline default function을 제공하고 싶을 때는 무조건 extension function을 사용해야한다. 이를 좀 더 명확하게 이해하기 위해서는 inline function에 대해서 이해가 필요하다.

---

- inline function이 왜 도입되었는가?

공식 문서내용을 직역해 보자면, 코틀린에서 고차함수를 사용하면 추가적인 메모리 할당 및 함수호출로 인한 runtime시 overhead가 발생한다. Inline functions는 내부적으로 함수 내용을 호출되는 위치에 복사하여, Runtime시의 overhead를 줄여준다.

<br/>
실제로 고차함수를 사용했을 때 어떠한 페널티가 존재하는지 확인해 보자. 

```kotlin
fun someMethod(a: Int, func: () -> Unit):Int {
    func()
    return 2*a
}

fun main(args: Array<String>) {
    var result = someMethod(2, {println("Just some dummy function")})
    println(result)
}
```

someMethod안에는 유닛값을 반환하는 func가 arg로 자리하게 되는데, 이를 자바로 변환해보면 다음과 같아진다.

```java
public final class InlineFunctions {
   public static final InlineFunctions INSTANCE;

   public final int someMethod(int a, @NotNull Function0 func) {
      func.invoke();
      return 2 * a;
   }

   @JvmStatic
   public static final void main(@NotNull String[] args) {
      int result = INSTANCE.someMethod(2, (Function0)null.INSTANCE);
      System.out.println(result);
   }

   static {
      InlineFunctions var0 = new InlineFunctions();
      INSTANCE = var0;
   }
}
```

차이가 보인다..! 자바로 변환했을 때는 someMethod를 호출하기 위해 싱글턴 객체를 생성하고 있음을 알 수 있었다.
> 즉, 내부적으로 객체 생성과 함수 호출을 하도록 구현이 되어있으며, 이런 부분이 성능을 떨어 뜨릴 수 있는 것이다.

그렇다면 inlinefunction으로 구성해보자. 

```kotlin
inline fun someMethod(a: Int, func: () -> Unit):Int {
    func()
    return 2*a
}
```

실제로 이 코드가 컴파일될 때, 컴파일러는 함수 내부의 코드를 호출하는 위치에 복사하게 된다. 물론 컴파일되는 바이트 코드의 양은 많아지겠지만, 함수 호출을 하거나 추가적인 객체를 생성하는 부분이 사라진다고 한다. 그럼 이를 자바로 컴파일 해보자.

```kotlin
public final class InlineFunctions {

   @JvmStatic
   public static final void main(@NotNull String[] args) {
      int a = 2;
      int var5 = false;
      String var6 = "Just some dummy function";
      System.out.println(var6);
      int result = 2 * a;
      System.out.println(result);
   }

}
```

정말 호출된 부분에서만 다시 컴파일되고 있었다. 하지만 이렇게 사용할 경우 단점또한 존재한다. 내부적으로 코드를 복사하기 때문에 인자로 전달받은 함수는 다른 함수로 전달되거나 참조될 수 없다는 것이다. 따라서 다음과 같은 코드는 컴파일 오류가 발생한다.

```kotlin
inline fun newMethod(a: Int, func: () -> Unit, func2: () -> Unit) {
    func()
    someMethod(10, func2)
}

fun someMethod(a: Int, func: () -> Unit):Int {
    func()
    return 2*a
}

fun main(args: Array<String>) {
    newMethod(2, {println("Just some dummy function")},
            {println("can't pass function in inline functions")})
}
```

이유는 `someMethod(10,func2)` 부분이다. inline 함수에서 인자로 전달받은 함수는 참조할 수 없기 때문에 func2를 전달하는것 또한 불가능하다.
> 물론 이 방법을 우회하는 방법도 존재한다. func2앞에 noinline키워드를 붙혀주면 된다.

noinline을 붙혔다고 할때, 자바로 컴파일 해보면 확인할 수 있다.

```kotlin
public final void newMethod(int a, @NotNull Function0 func, @NotNull Function0 func2) {
   func.invoke();
   this.someMethod(10, func2);
}

public final int someMethod(int a, @NotNull Function0 func) {
   func.invoke();
   return 2 * a;
}

@JvmStatic
public static final void main(@NotNull String[] args) {
   String var6 = "Just some dummy function";
   System.out.println(var6);
   this_$iv.someMethod(10, func2$iv);
}
```

실제로 func2()를 제외한 나머지 코드들은 inline처리되어 복사되었다.

> 정리해보자. inline키워드를 사용하여 함수를 정의하면 런타임 오버헤드를 줄일 수 있다. 하지만 반대로 코드 양이 많은 함수를 인라인 처리하면 바이트코드 양이 많아질 수 있고 이는 최적화를 방해할 수 있다. 따라서 짧은 길이의 함수일때 사용하는 것이 유리하다.

---

다시 본론으로 돌아와보자. 위의 인라인함수 특징을 이해했다면 이는 반드시 final이라는 속성을 갖는다. 하지만 interface에서 final이란 있을 수 없는 것이다. 하지만 확장함수로 이러한 인라인 함수를 정의한다면 사용할 수 있다. 

<img src="/assets/img/kotlin_study/2.png">

> 사실 이부분은 직접 프로젝트하며 써본적은 없어서 크게 와닿지 않는다.. 여러번 확인해가며 보아야겠다.

#### mandatory한 상황 - Iteration을 주고 싶을때

만약 내가 정의한 클래스에 `for (item in collection) { ... }`와 같은 기능을 주고 싶을때 사용할 수 있다. 바로 해당 클래스이름뒤에 `iterator()` 익스텐션을 주어 Iterator<>를 정의하는 방법이다. 당연히 iteration을 실행하기 위해서는 next()와 hasnext()에 대한 구현부를 필요로 한다.

#### mandatory한 상황 - operator 정의

operator에 대한 새로운 정의란 다음과 같은 상황이 있다. 

```kotlin
data class Point(val x: Int, val y: Int)

operator fun Point.unaryMinus() = Point(-x, -y)

val point = Point(10, 20)

fun main() {
   println(-point)  // prints "Point(x=-10, y=-20)"
}
```

인스턴스.unaryPlus(), unaryMinus(), not()을 새롭게 정의하여 +,-,!가 붙었을 때의 새로운 정의를 해줄 수 있다.
> ++, --의 경우는 inc(), dec() / +,-,* ... 의 경우는 plus(),minus(),times()등이 존재한다. [공식 문서](https://kotlinlang.org/docs/operator-overloading.html#binary-operations)를 참조하면 더 많은 정보를 확인할 수 있다.