---
title: "OnBackPressed() Deprecated, 어떻게 할까?"
date: 2025-04-02 16:47:59 +0900
categories: [Android]
tags: [OnBackPressed, Deprecated, onBackPressedDispatcher, onBackPressedCallback]
---

## onBackPressed() 함수란?
`Compose` 로 개발되었거나 `View` 방식으로 개발되었더라도 최근에 개발된 안드로이드 코드를 접했다면 보기 힘든 함수이다.

물론 최근에는 `Compose` 로 대부분 개발하는 추세이기 때문에 위 함수를 아예 접할 기회가 없을지도 모르지만,

어느 회사던 Lagacy 한 프로젝트는 남아있기 마련이고, 이러한 프로젝트를 유지보수 해야할 일도 빈번하다.

`onBackPressed()` 함수는 사용자가 네비게이션 바에 위치한 "시스템 뒤로가기" 버튼을 클릭했을 때 혹은 뒤로가기 제스처를 통해 호출되는 함수이다.

우리는 이 함수를 오버라이드 하여 뒤로가기가 눌렸을때의 동작을 변경 할 수 있었다.
```kotlin
class MainActivity : AppCompatActivity() {
    override fun onBackPressed() {
        // 시스템 백 버튼 눌렀을때의 동작 정의
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        
        ...
    }    
}
```
이를 통해 할 수 있는 가장 쉬운 예시로, 뒤로가기 버튼을 눌렀을떄 종료를 확인하는 팝업이 나타난다던지, "한번 더 누르면 앱이 종료됨" 을 알리는 `Toast` 를 띄우는 등으로 사용이 가능하다.

<p align="left">
  <img src="https://host.ggoggo.duckdns.org/Blog/250402_onBackPressed_deprecated/backpress_ex1.png" width="45%">
  <img src="https://host.ggoggo.duckdns.org/Blog/250402_onBackPressed_deprecated/backpress_ex2.png" width="45%">
</p>

## onBackPressed() 의 Deprecated
그러나, 이 함수는 **API Level 33** 부터 `Deprecated` 되었다.

![](https://host.ggoggo.duckdns.org/Blog/250402_onBackPressed_deprecated/deprecated_docs.png)

[onBackPressed Deprecated 공식 문서](https://developer.android.com/reference/android/app/Activity#onBackPressed())

설명을 보면, **onBackPressed 는 API Level 33 에서 Deprecated 되었으므로**, 네비게이션의 Back 버튼에 대한 동작을 처리하기 위해 `OnBackInvokedCallback` 혹은 `androidx.activity.OnBackPressedCallback` 를 사용하라고 되어있다.

`OnBackInvokedCallback` 의 경우, **API Level 33 이상을 필요로 하므로**, 이전 버전과의 호환을 위해 `OnBackPressedCallback` 를 사용하는 방법에 대해서 알아보자.

_(대부분의 경우에서 제작하는 앱이 Android 33 이상부터 지원하도록 되어있는 앱은 거의 없을 것이라 생각한다)_

## OnBackPressedCallback 내부 동작
기본적으로 프로젝트를 생성하면, Activity 는 AppCompatActivity 를 상속받는다.

AppCompatActivity 는 아래와 같은 상속 구조를 가지고 있다

```
MainActivity (com.example.myproject)
    └─  AppCompatActivity (androidx.appcompat.app)
        └─ FragmentActivity (androidx.fragment.app)
            └─ ComponentActivity (androidx.activity)
                └─ ComponentActivity (androidx.core.app)
                    └─ Activity (android.app)
```
사용하라고 명시된 `OnBackPressedCallback` 은 `androidx.activity` 패키지에 위치한 추상 클래스이다.

`OnBackPressedCallback` 인터페이스는 `OnBackPressedDispatcher` 의 `onBackPressed` 를 처리한다.

```kotlin
abstract class OnBackPressedCallback(enabled: Boolean) {
    /**
     * The enabled state of the callback. Only when this callback
     * is enabled will it receive callbacks to [handleOnBackPressed].
     *
     * Note that the enabled state is an additional layer on top of the
     * [androidx.lifecycle.LifecycleOwner] passed to
     * [OnBackPressedDispatcher.addCallback]
     * which controls when the callback is added and removed to the dispatcher.
     */
    @get:MainThread
    @set:MainThread
    var isEnabled: Boolean = enabled
        set(value) {
            field = value
            enabledChangedCallback?.invoke()
        }
        
    ...
    
        /**
     * Callback for handling the [OnBackPressedDispatcher.onBackPressed] event.
     */
    @MainThread
    abstract fun handleOnBackPressed()
    
    ...
}
```

여러 메서드들이 정의되어 있지만, 주요하게 쓸 2개의 구성요소는 **`isEnabled`** 와 **`handleOnBackPressed()`** 다.

이를 사용하기 위해서는 androidx.activity 패키지 `ComponentActivity` 의 `OnBackPressedDispatcher.addCallback()` 을 통해 `OnBackPressedCallback` 을 등록 시켜야 한다.

그럼 시스템 백 버튼이 눌렸을때 `OnBackPressedDispatcher` 는 등록한 `OnBackPressedCallback` 중 `isEnabled` 가 `true` 라면 `Callback` 으로 등록한 동작을 실행시키고, `false` 라면 백 버튼 눌렸을때 기본적으로 수행하는 동작을 실행하게 된다.

```kotlin
class OnBackPressedDispatcher constructor(
    private val fallbackOnBackPressed: Runnable?,
    private val onHasEnabledCallbacksChanged: Consumer<Boolean>?
) {
    private val onBackPressedCallbacks = ArrayDeque<OnBackPressedCallback>()
   
    ...
   
    @MainThread
    fun onBackPressed() {
        val callback = onBackPressedCallbacks.lastOrNull { // isEnabled 가 true 인 마지막 callback 르 찾는다
            it.isEnabled
        }
        inProgressCallback = null
        if (callback != null) { // callback 이 null 이 아니라면 등록한 callback의 handleOnBackPressed() 를 실행
            callback.handleOnBackPressed()
            return
        }
        fallbackOnBackPressed?.run() // callback 이 null 이라면(isEnabled 가 false 일 경우) 기본 동작 실행
    }
    
    ...
    
}
```

내부 동작을 정리하면

1. 사용자가 시스템 백 버튼 누름!
2. `OnBackPressedDispatcher` 의 `onBackPressed()` 실행
3. `ArrayDeque` 타입으로 저장된 `OnBackPressedCallback` 중에서 `isEnabled` 가 `true` 인 마지막 Callback 을 찾음
4. `Callback` 이 `null` 이 아닐 경우(`isEnabled` 로 지정된 `Callback`) 을 실행한다!

가 된다.

이렇게 내부동작을 살펴봤는데, 결론적으로 개발자가 해야할 일은

1. 시스템 백 버튼을 눌렀을 때 실행할 동작을 `OnBackPressedCallback` 익명 객체로 선언한다.
2. Activity 단의 onCreate 에서 `onBackPressedDispatcher` 객체를 가져와 `addCallback` 시켜준다.
3. 뒤로 버튼을 누르며 원하는 동작이 실행되는지 관찰한다. 

## 한번 해보자
```kotlin
class MainActivity : AppCompatActivity() {

    // 1. OnBackPressedCallback 정의
    private val backPressedCallback = object : OnBackPressedCallback(true) {
        override fun handleOnBackPressed() {
            // 2. back 버튼을 누르면 실행할 동작 작성
        }
    }
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        
        // 3. onBackPressedDispatcher 의 addCallback 으로 OnBackPressedCallback 등록
        onBackPressedDispatcher.addCallback(this, backPressedCallback)
        
        ...
    }
}
```

생각보다 간단하다. 

위와 같이 하면, 백 버튼을 눌렀을 경우 `backPressedCallback` 의 `handleOnBackPressed()` 에 정의한 동작이 실행된다.

### 내가 정의한 동작과 기본적인 동작을 조건에 따라 실행하고 싶을 경우

어떠한 경우에서는 원래 동작과, 내가 정의한 동작을 조건에 따라 실행하고 싶을 경우도 있을 것이다.

가령, 필자가 봤던 경우는 `BottomNavigation` 에 5개의 `Fragment` 가 있지만, 홈 `Fragment` 가 아닌 곳에서는 `Back 버튼`을 누르면 기본 동작인 `fragment pop` 이 동작하고, `홈 Fragment` 에서는 종료 확인 팝업을 눌러줘야 하는 경우가 있었다.

그런 경우에는 아래와 같이 할 수 있다.
```kotlin
class MainActivity : AppCompatActivity() {

    private val backPressedCallback = object : OnBackPressedCallback(true) {
        override fun handleOnBackPressed() {
            if(navController.currentDestination!!.id == R.id.navigation_home) {
                // 종료 확인 팝업 띄워줌
            } else { // 원래 동작 수행해야 함
                isEnabled = false // 이 콜백의 isEnabled 를 false 로 바꿔줌. 
                // OnBackPressedDispatcher 의 onBackPressed() 를 강제로 실행
                onBackPressedDispatcher.onBackPressed()
                // 이렇게 하면 시스템 백버튼 클릭 시 OnBackPressedDispatcher 에서 isEnabled 인 callback 이 존재하지 않게 되므로
                // 기본 동작이 수행됨
            }
        }
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        onBackPressedDispatcher.addCallback(this, backPressedCallback)
    }
}
```

## 여러개의 OnBackPressedCallback 을 등록했을 경우

결론부터 말하자면, 마지막에 `addCallback` 을 통해 추가한 `OnBackPressedCallback` 만 동작하게 된다.

`OnBackPressedDispatcher` 내부에서 `OnBackPressedCallback` 은 `ArrayDeque` 로 관리되며, `onBackPressed()` 시
`isEnabled` 가 `true` 인 `callback` 중 제일 마지막의 `handleOnBackPressed()` 를 실행하게끔 설계되어 있기 때문이다.

이런 경우가 있을진 모르곘지만, 만약 백 버튼 눌렀을때 단 한번만 동작할 행동이 존재한다면
1. 기본적으로 동작할 `OnBackPressedCallback` 을 정의
2. 단 한번만 동작할 `OnBackPressedCallback` 을 정의. `handleOnBackPressed()` 내에 동작을 정의하고 마지막에 `remove()` 를 추가하여 `callback` 을 지워줄 수 있다.
3, 두개의 `Callback` 을 추가(이때, 먼저 실행해야 할 `Callback` 을 나중에 추가한다)
4. 백 버튼을 누르면, 한번만 동작할 `OnBackPressedCallback` 이 실행되고, remove 를 통해 `OnBackPressedDispatcher` 내의 `ArrayDeque` 에서 지워진다.
5. 한번 더 누르면 기본적으로 동작할 `OnBackPressedCallback` 이 실행된다.

코드로 보면 다음과 같다.
```kotlin
class MainActivity : AppCompatActivity() {

    private val backPressedCallback1 = object : OnBackPressedCallback(true) {
        override fun handleOnBackPressed() {
            Log.d("glion", "백 버튼 누르면 기본적으로 실행할 동작")
        }
    }

    private val backPressedCallback2 = object : OnBackPressedCallback(true) {
        override fun handleOnBackPressed() {
            Log.d("glion", "2번째로 추가한 콜백")
            remove()
        }
    }

    private val backPressedCallback3 = object : OnBackPressedCallback(true) {
        override fun handleOnBackPressed() {
            Log.d("glion", "3번째로 추가한 콜백")
            remove()
        }
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        onBackPressedDispatcher.addCallback(this, backPressedCallback1)
        onBackPressedDispatcher.addCallback(this, backPressedCallback2)
        onBackPressedDispatcher.addCallback(this, backPressedCallback3)
    }
}
```
![](https://host.ggoggo.duckdns.org/Blog/250402_onBackPressed_deprecated/multi_backpressed.png)

백 버튼을 눌렀을 경우, 마지막에 추가된 `backPressedCallback3` 부터 `backPressedCallback2`, `backPressedCallback1` 순으로 실행되는 것을 확인할 수 있다.

## 결론
`onBackPressed()` 함수를 오버라이딩 하는 방식은 이미 **`Deprecated`** 된지 오래다.

혹시 레거시 프로젝트에 아직 `onBackPressed()` 를 오버라이딩 하여 사용하고 있다면, `OnBackPressedCallback` 과 `onBackPressedDispatcher` 로 변경해보자.