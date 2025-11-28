---
title: 'SAA & Fragment Navigation - 1 (XML Based Graph'
date: 2024-12-23 11:30:00 +0900
categories: [Android, Jetpack]
tags: [jetpack, fragment, navigation, xml, graph, SAA]
---

### SAA?
Fragment Navigation 에 대해 다루기 이전에 SAA 의 개념에 대한 이야기이다.

지금은 전부 Compose 방식으로 넘어갔지만, Compose 등장 이전 Xml 방식으로 UI 를 작성하고 Activity, Fragment 들로 구성하였을때 등장한 개념이다.

**Single Activity Architecture** 의 약자로서, 앱 내에 1개 혹은 아주 적은 수의 Activity 만을 두고 나머지 화면을 Fragment 로 구성하는 방식이다.

Activity 로 구성할 경우 화면 전환이 Fragment 에 비해 비교적 간편하다고 느낄 수 있는데, 왜 이런 방법을 써야 할까?

Fragment 는 Activity 보다 가볍기 때문에 동일한 화면 Flow 를 Activity 만으로 구성하는 것보다 Fragment 로 구성하는 것이 훨씬 가볍다.

이는 각 Activity 는 다른 프로세스에서 실행되는데 반해, **Fragment 는 동일한 Activity 의 메모리 영역에서 움직이는 것이라 리소스 소모가 적기 때문이다.**

동일한 메모리 영역에서 동작하기 때문에 Activity Scope 내에서 SharedViewModel 을 사용하여 데이터를 공유 및 전달하는 것 또한 가능하다.

또한, Navigation 을 잘 구성해두면, 간편하다고 느낀 Intent 와 startActivity 등의 화면 전환 방법보다 더욱 간편하게 이용이 가능하고, 화면 전환 Animation 또한 비교적 간편하게 추가가 가능하다.

그러나, 장점만 있는것은 아니고 Fragment 또한 각 LifeCycle 이 존재하므로 더 복잡해 질 수 있으며 프로젝트 설계 단계에서 더 많은 부분을 고려해야 할 것이다.

> 이전에 Activity 기반으로 앱을 작성헀다면, SAA 와 다음으로 설명할 Fragment Navigation 을 사용하여 Fragment 기반으로 앱을 작성해보자.
{: .prompt-tip }

### Navigation
Android Jetpack 의 Navigation 구성요소에는 Navigation 라이브러리, Fragment 간 데이터를 전달하기 위한 Safe Args Gradle 플러그인, 화면 전환을 구현하는데 도움이 되는 여러 도구들이 포함되어 있다.

아래는 Navigation 의 주요 개념과 이를 구현하는데 사용하는 기본 유형을 설명한다 (Compose 를 사용하던, View 방식의 UI 를 사용하던 이러한 개념이 적용되지만, 사용하는 구체적인 방법은 다를 수 있다)

이 글에서는 View 방식의 UI 를 사용할 경우, 그중 xml 방식의 graph 를 생성하여 사용하는 방식을 설명한다.

- **Host**
  - 전체 Navigation 대상이 포함된 UI 요소.
  - 레이아웃 내에서 독립적인 탐색이 발생할 수 있는 영역이다.
- **Graph**
  - 앱 내의 모든 Navigation 대상과 연결 방법을 정의하는 데이터 구조이다.
  - 이 글에서는 xml 기반의 graph 를 생성하여 사용하는 것을 다룬다.
- **Controller**
  - Fragment 간 Navigation 을 관리한다.
  - Controller 는 Navigation 간 탐색, 딥 링크 처리, 백스택 관리 등의 작업을 위한 메서드를 제공한다.
- **Destination**
  - Navigation Graph 의 노드로서, 이동할 화면이다.
  - Controller 를 통해 특정 Destination 으로 이동할 경우 Host 에 UI 컨텐츠가 표시된다.
- **Route**
  - 대상과 필요한 데이터를 식별할 수 있다.
  - Route 를 사용하여 탐색할 수 있으며 이를 사용하여 특정 Destination 으로 이동이 가능하다. Route 는 직렬화 가능한 데이터 유형이여야 한다.

**Navigation 을 사용하게 되면 아래와 같은 장점을 얻을 수 있다.**
- FragmentManager 의 Transaction 의 처리를 간단하게 할 수 있다.
- 화면의 뒤로가기 혹은 위로가기의 동작을 올바르게 처리해준다.
- 화면 전환 애니메이션 처리에 대한 표준화된 리소스를 제공해준다.
- 딥 링크를 구현하고 처리할 수 있다.
- fragment 간 데이터를 전달해야 할 경우, safe-args 를 사용하여 안정적으로 데이터 전달이 가능하다.
- BottomNavigation 과 같은 화면 이동 패턴에 대해 최소한의 추가작업으로 구현이 가능하도록 한다.
- viewModel 의 범위를 Navigation Graph 로 지정하여 그래프 간 UI 데이터를 공유할 수 있다.

### 추가하기
프로젝트에 Jetpack Navigation 을 포함하려면 앱 수준 build.gradle 에 아래의 내용을 추가하여 시작할 수 있다.

```gradle
plugins {
    // 추후 DSL 방식을 사용하여 graph 를 코드로 생성하고 Route 를 통해 화면 이동을 하고자 할때, 사용될 코틀린 직렬화 플러그인.
    // 설명할 xml 방식의 graph 를 사용할 경우 추가하지 않아도 된다.
    kotlin("plugin.serialization") version "2.0.21"
}

dependencies {
    val nav_version = "2.8.4" // 구글 공식문서 기준 버전.

    // 추가해야할 필수 항목
    implementation("androidx.navigation:navigation-fragment:$nav_version")
    implementation("androidx.navigation:navigation-ui:$nav_version")

    // Feature module support for Fragments
    implementation("androidx.navigation:navigation-dynamic-features-fragment:$nav_version")

    // 테스트코드에서 Android Navigation 을 사용하기 위함. 필수가 아니다.
    androidTestImplementation("androidx.navigation:navigation-testing:$nav_version")

    // JSON 직렬화 라이브러리. xml 방식의 graph 를 사용할 경우 추가하지 않아도 된다.
    implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.7.3")
}
```

### 화면 구성
샘플 코드의 화면 구성은 6개의 화면으로 구성되어있고, 아래와 같이 진행될 것이다.

![](https://host.ggoggo.duckdns.org/Blog/241223_navigation/flow.png)

먼저, 프로젝트를 생성하고 res 폴더를 우클릭하여 `new` -> `Android Resource Directory` -> Resource type 에 `navigation` 를 선택하고 OK 를 누른다.
Directory name 은 navigation 을 선택하게 되면 자동으로 지정된다.

![](https://host.ggoggo.duckdns.org/Blog/241223_navigation/add_navigation_folder_1.png)
![](https://host.ggoggo.duckdns.org/Blog/241223_navigation/add_navigation_folder_2.png)

그 뒤 생성된 navigation 폴더에서 `new` -> `Navigation Resource File` 하여 xml 기반의 graph 를 생성한다.

이제 graph 에 진행될 화면에 대한 Fragment 를 추가해 주어야 한다.

미리 생성해둔 Fragment 가 존재한다면, 목록에서 추가할 Fragment 를 클릭하여 추가해주면 되고, 없다면 `Create New Destination` 을 눌러 새로운 Fragment 를 추가할 수 있다.

![](https://host.ggoggo.duckdns.org/Blog/241223_navigation/add_fragment_in_graph.png)

이런식으로 추가할 화면들에 대해 전부 Graph 에 넣어준다면, 아래와 같이 보일 것이다.

![](https://host.ggoggo.duckdns.org/Blog/241223_navigation/complete_add_graph.png)

화면 이동에 대한 Action 을 추가할 때는, 추가한 Fragment 의 우측 중앙에 O 을 클릭하여 이동하고자 하는 Fragment 로 화살표를 연결해주기만 하면 Action 에 대한 xml 스크립트가 자동으로 생성된다.

![](https://host.ggoggo.duckdns.org/Blog/241223_navigation/graph_add_action.png)

Fragment 를 추가하고, Action 을 추가하면 좌측에 여러 코드들이 자동으로 생성되는데, 세부 내용은 주석으로 설명하겠다.

```xml
<navigation xmlns:android="[http://schemas.android.com/apk/res/android](http://schemas.android.com/apk/res/android)"
    xmlns:app="[http://schemas.android.com/apk/res-auto](http://schemas.android.com/apk/res-auto)"
    xmlns:tools="[http://schemas.android.com/tools](http://schemas.android.com/tools)"
    android:id="@+id/nav_graph"
    app:startDestination="@id/fragmentHome"> <fragment
        android:id="@+id/fragmentHome" 
        android:name="com.example.navigationfragmentsample.graph.FragmentGraphHome" 
        android:label="fragment_home"
        tools:layout="@layout/fragment_graph_home" >
        <action
            android:id="@+id/action_fragmentHome_to_fragmentOption1" 
            app:destination="@id/fragmentOption1" 
            app:enterAnim="@anim/slide_in" 
            app:exitAnim="@anim/nav_default_exit_anim" 
            app:popEnterAnim="@anim/nav_default_pop_enter_anim" 
            app:popExitAnim="@anim/slide_out" /> 
            <action
            android:id="@+id/action_fragmentHome_to_fragmentOption2"
            app:destination="@id/fragmentOption2"
            app:enterAnim="@anim/slide_in"
            app:exitAnim="@anim/nav_default_exit_anim"
            app:popEnterAnim="@anim/nav_default_pop_enter_anim"
            app:popExitAnim="@anim/slide_out" />
    </fragment>
</navigation>
```

이제 각 화면을 추가하고 Action 을 추가했다면, Navigation 을 통해 Fragment 전환할 준비가 어느정도 마무리되었다.

xml 방식으로 Graph 를 생성하는 방식 외에도 Kotlin DSL 을 통해 프로그래매틱 방식으로 Navigation 그래프를 만들 수 있다.
이는 XML 리소스 파일을 사용하는 것보다 더 깔끔하고 현대적인 방법이라고 공식문서에서 소개하고 있다(실제 Compose 에서 Navigation 을 사용하는 방법과 매우 비슷하다)

이 방식은 다음 포스팅에서 알아본다.

### 화면 이동
graph 를 만들었다면 이제 fragment 를 띄울 Host 를 지정해주어야 한다.

MainActivity 의 레이아웃에 다음과 같이 추가한다.

```xml
<androidx.constraintlayout.widget.ConstraintLayout
    android:id="@+id/main"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">
    <androidx.fragment.app.FragmentContainerView
        android:id="@+id/nav_main"
        android:name="androidx.navigation.fragment.NavHostFragment"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintBottom_toBottomOf="parent"
        app:defaultNavHost="true"
        app:navGraph="@navigation/nav_graph"/>
</androidx.constraintlayout.widget.ConstraintLayout>
```
여기서 중요한 것은, `android:name="androidx.navigation.fragment.NavHostFragment"` 로 지정하여 FragmentContainerView 를 Host 로 설정해주고,

생성한 graph 를 `app:navGraph="@navigation/nav_graph"` 로 지정해 주는 것, 그리고 `app:defaultNavHost="true"` 를 추가해주는 것이다.

`app:navGraph` 는 생성한 graph 를 사용하는 Host 라는 것을 알려주는 것이고, `app:defaultNavHost="true"` 는 시스템 뒤로가기 버튼을 가로채 backPressed 동작 시 navigateUp() 동작을 수행하도록 해준다.

`app:navGraph`, `app:defaultNavHost` 두가지 속성은 자동완성이 되지 않으므로, 오타 없이 기입하여 빌드 할 수 있도록 한다.

우선적으로 이렇게 해두면, 빌드했을때 startDestination 으로 지정한 화면이 맨 처음에 나타나게 된다.

**이제, startDestination 인 화면 1에서 화면 2로 이동하는 방법을 알아보자.**

화면 이동은 NavController 를 사용하여 수행할 수 있다.

fragment 에서 `findNavController` 를 통해 부모 Activity 에 선언한 NavHost 의 Controller 를 가져올 수 있고, navController 의 `Maps` 를 통해 정의해준 Action 을 실행할 수 있다.

일반적으로 NavController 는 Fragment 에서 사용되기에 Fragment 에서 NavController 는 `findNavController()` 를 통해 가져올 수 있지만, Activity 에서 NavController 를 가져오는 방법은 조금 다르다.

Activity 내에서 `supportFragmentManager.findFragmentById(FragmentContainer 의 id) as NavHostFragment` 를 통해 Host 를 가져온 뒤 `NavHostFragment.navController` 를 통해 NavController 를 가져올 수 있다.

아래 코드를 보자

```kotlin
with(binding) {
    btnOption1.setOnClickListener {
        findNavController().navigate(R.id.action_fragmentHome_to_fragmentOption1)
    }
    btnOption2.setOnClickListener {
        findNavController().navigate(R.id.action_fragmentHome_to_fragmentOption2)
    }
}
```

화면에 버튼이 2개가 있고, btnOption1 을 클릭하게 되면

`findNavController.navigate(R.id.actionId)` 의 형태로 정의해둔 Action 을 수행할 수 있고 사전에 정의해둔 애니메이션이 동작하며 화면이 이동하게 된다.

### 화면 간 데이터 이동
Safe Args Gradle 플러그인을 사용한다. 이 플러그인은 Android View 와 Fragment 에서만 사용할 수 있다.

<br>

먼저 앱 수준의 build.gradle 파일에 다음의 classpath 를 포함시킨다.

```gradle
// buildscript 부분이 아예 존재하지 않을 경우, 전부 추가해준다.
buildscript {
    dependencies {
        classpath("androidx.navigation:navigation-safe-args-gradle-plugin:2.8.4")
    }
}
plugins {
    alias(libs.plugins.android.application) apply false
    alias(libs.plugins.jetbrains.kotlin.android) apply false
}
```

<br>

또한 app 수준 build.gradle 에서 두가지 플러그인 중 하나를 적용해준다.
- 자바 모듈 혹은 자바와 Kotlin이 섞인 모듈의 경우
  ```gradle
  plugins {
    id("androidx.navigation.safeargs")
  }
  ```
- Kotlin 전용 모듈의 경우
  ```gradle
  plugins {
    id("androidx.navigation.safeargs.kotlin")
  }
  ```

오류가 생긴다면 gradle.properties 파일에 `android.useAndroidX=true` 가 있는지 확인한다.

플러그인을 추가했다면, graph 를 열어 데이터를 받고자 하는 Fragment 의 Attributes 에서 Arguments 를 추가해준다.

![](https://host.ggoggo.duckdns.org/Blog/241223_navigation/fragment_arguments.png)

전달 가능한 데이터 타입은 기본적인 primitive Type 인 Integer, Float, Long, Boolean 과 String, 그리고 직렬화 가능한 Object 이다.

파라미터 이름, 타입, 배열 여부, Null 여부, 기본값을 설정한 뒤 Add 를 누르게 되면 자동으로 아래와 같이 추가된다.

```xml
<argument
    android:name="number" 
    app:argType="integer"
    app:nullable="true"
    android:defaultValue="@null" />
    ```

<br>

데이터를 전달하려는 Fragment 에서 navigate 시 argument 에 정의한 데이터를 전달할 수 있다.

이 경우, 라이브러리를 통해 빌드된 `Fragment이름Directions` 형태의 NavDirection 객체에서 action 을 불러와 argument 전달할 데이터를 넣어주면 된다.

<br>

정의한 그래프가 다음과 같을때,
```xml
<fragment
    android:id="@+id/fragmentOption1"
    android:name="com.example.navigationfragmentsample.graph.FragmentGraphOption1"
    android:label="fragment_option1"
    tools:layout="@layout/fragment_graph_option1" >
    <action
        android:id="@+id/action_fragmentOption1_to_fragmentResult"
        app:destination="@id/fragmentResult"
        app:enterAnim="@anim/slide_in"
        app:exitAnim="@anim/nav_default_exit_anim"
        app:popEnterAnim="@anim/nav_default_pop_enter_anim"
        app:popExitAnim="@anim/slide_out" />
</fragment>

<fragment
    android:id="@+id/fragmentResult"
    android:name="com.example.navigationfragmentsample.graph.FragmentGraphResult"
    android:label="fragment_result"
    tools:layout="@layout/fragment_graph_result" >
    <argument
        android:name="number"
        app:argType="integer"
        app:nullable="false" />
</fragment>
```
FragmentGraphOption1 에서 FragmentGraphResult 로 데이터를 전달할 경우
```kotlin
val willSendNumber = 14
val action = FragmentGraphOption1Directions.actionFragmentOption1ToFragmentResult(willSendNumber)
findNavController().navigate(action)
```
와 같이 할 수 있다.

FragmentGraphResult 에서 전달된 데이터를 받을때는 bundle 을 통해 Fragment 로 전달된 데이터를 받을때처럼 arguments 를 이용한다.
```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    arguments?.let {
        val data = it.getInt("number") // graph 에 생성한 argument 의 name
    }
}
```

navigate 함수의 내부를 살펴보면
```kotlin
@MainThread
public open fun navigate(@IdRes resId: Int, args: Bundle?, navOptions: NavOptions?) {
    navigate(resId, args, navOptions, null)
}
```
로 되어있는데, 전달된 args 가 bundle 타입으로 되어있는것을 알 수 있다.

그렇다면, Data 를 전달할때도 bundle 을 통해 직렬화 가능한 데이터 타입을 전달할 수 있고, 동일하게 받을 수 있을 것이다.

### 샘플 프로젝트
해당 글을 작성하며 사용된 샘플 프로젝트이다.

graph_dsl_on_activity 브랜치의 NavigationFragmentSample 폴더 내 graph 패키지에서 위의 내용을 확인 할 수 있다.

[샘플 프로젝트](https://github.com/GlionDev/AndroidStudy/tree/graph_dsl_on_activity)



### 참고 링크

- [SAA with jetpack Navigation](https://no-dev-nk.tistory.com/99)
- [Android 공식 Docs - 앱 탐색](https://developer.android.com/guide/navigation/principles?hl=ko)
- [Jetpack Navigation Animation](https://philosopher-chan.tistory.com/1501)
- [안드로이드(Android) - Navigation 사용법](https://jae-young.tistory.com/46)