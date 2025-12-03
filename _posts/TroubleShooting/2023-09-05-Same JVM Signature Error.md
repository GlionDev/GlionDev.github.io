---
title: "Platform declaration clash: The following declarations have the same JVM signature 에러"
date: 2023-09-05 16:00:11 +0900
categories: [TroubleShooting]
tags: [Kotlin, TroubleShooting, JVM]
---

## 문제 발생
Kotlin 의 범위함수 `apply` 에 대해 공부하면서 예제코드를 작성할 때였다.

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(R.layout.activity_main)
    
    applyFunction()
}

inner class Person(){
    var name: String?= null
    var age: Int? = null

    fun getName(): String?{
        return name
    }
}

private fun applyFunction(){
    val person = Person().apply{
        name = "ApplyTest"
        age = 1
    }.getName()
    Log.d("Glion", person?: "Null")
}
```
applyFunction() 호출 시 범위함수 apply를 사용하여 person 객체의 속성을 변경하고, getName을 호출하여 person 변수에 name이 저장되게끔 하였다. 그리고 로그를 찍어 확인해보려고 했으나

> Platform declaration clash: The following declarations have the same JVM signature fun `<get-name>`(): String? defined in com.example.oddmentsproject.activity.MainActivity.Person fun getName(): String? defined in com.example.oddmentsproject.activity.MainActivity.Person

와 같은 오류가 나타났다.

구글링을 해보니, `<get-name>()` 과 `getName` 이 같은 `JVM Signature` 를 가지고 있다는 것이다.  
코틀린코드가 컴파일 될 때 자바 바이트코드로 변경되는데, 이때 이 `JVM Signature` 가 동일하기 때문에 오류가 난다는 말인데...

내가 짠 코드에서 `getName` 함수는 하나뿐이고, 같은 `JVM Signature` 를 가지고 있다고 하는데 무슨 말인지도 잘 모르겠다.

## 원인 분석
로그를 보면, Person 클래스의 `getName()` 과 `<get-name>` 이 중복되었다고 하고, 바이트 코드로 변경될때 동일한 이름이 있기 때문에 오류가 난다고 했으니, 위 코드를 자바 바이트 코드로 디컴파일해서 살펴보자.
```java
   private final void applyFunction() {
      Person var2 = new Person();
      int var4 = false;
      var2.setName("ApplyTest");
      var2.setAge(1);
      String person = var2.getName();
      String var10001 = person;
      if (person == null) {
         var10001 = "Null";
      }

      Log.d("Glion", var10001);
   }
   
      public final class Person {
      @Nullable
      private String name;
      @Nullable
      private Integer age;

      @Nullable
      public final String getName() { // 이부분과
         return this.name;
      }

      public final void setName(@Nullable String var1) {
         this.name = var1;
      }

      @Nullable
      public final Integer getAge() {
         return this.age;
      }

      public final void setAge(@Nullable Integer var1) {
         this.age = var1;
      }

      @Nullable
      public final String getName() { // 이부분이 동일하다
         return this.name;
      }
   }
```
이 코드는 위의 `Person` 클래스 생성부분과 `applyFunction()` 부분을 디컴파일 한 코드이다.

`Person` 클래스 생성부분을 보면, 멤버변수 `name` 과 `age` 가 `private` 으로 선언되어 있고, 이에 접근하기 위해 생성된 `getter`,`setter` 가 get*, set* 이라는 네이밍 규칙으로 생성된 것을 확인 할 수 있다. 그리고 내가 만든 동일한 `getName()` 함수도 존재한다.

_**즉, 동일한 함수가 2개 생성되어 컴파일 오류가 난 것이다**_

## 해결 방법
1. 자동생성되는 getter, setter 의 이름인 **"get/set + 변수명"** 의 네이밍 규칙을 피할 수 있도록 함수명을 작성한다.  
    제일 간단한 방법이다. 가령 `getName()` 함수명을 `getNameProperty()` 라고 한다던지...
2. `@JvmName` 을 붙여준다.
    중복된 `JVM Signature` 를 변경해주는 것이다. 내가 생성한 `getName()` 함수 위에 다음과 같이 추가해주면 된다.
    ```kotlin
    inner class Person(){
        var name: String?= null
        var age: Int? = null
        @JvmName("userDefinedGetName")
        fun getName(): String?{
            return name
        }
    }
    ```
    `@JvmName`을 사용하여 `userDefinedGetName`으로 `JVM Signature`를 변경해주었다. 디컴파일하면 다음과 같다.

    JVM 에서 컴파일 할때, `Kotlin` 으로 작성한 `getName` 이라는 함수는 `@JvmName` 어노테이션 때문에 `userDefinedGetName` 으로 변경된다.
    ```
    public final class Person {
        @Nullable
        private String name;
        @Nullable
        private Integer age;
        @Nullable
        public final String getName() {
            return this.name;
        }
        public final void setName(@Nullable String var1) {
            this.name = var1;
        }
        @Nullable
        public final Integer getAge() {
            return this.age;
        }
        public final void setAge(@Nullable Integer var1) {
            this.age = var1;
        }
        @JvmName(
            name = "userDefinedGetName"
        )
        @Nullable
        // getName() 으로 되어있던 함수명이 내가 지정해준 JvmName으로 변경되었다
        public final String userDefinedGetName() { 
            return this.name;
        }
    }
    ```

이로서, 클래스 생성 후 내부에 함수를 작성할때, 함수명에 유의해야 함을 알았다.

## 참고 URL
[JVM Signature error 대처하기 - 출처 : velog 블로그 _im_ssu.log ](https://velog.io/@_im_ssu/JVM-signature-error-%EB%8C%80%EC%B2%98%ED%95%98%EA%B8%B0)  
[Kotlin 에서 자주 사용하는 annotation 정리](https://codechacha.com/ko/kotlin-annotations/)