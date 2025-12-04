---
title: "onBackPressedDispatcher : Unresolved references"
date: 2025-04-06 14:01:36 +0900
categories: [TroubleShooting]
tags: [TroubleShooting, onBackPressedDispatcher, onBackPressed(), Conflict Dependencies]
---


## 들어가기 전에
해당 글은 `onBackPressedDispatcher` 를 사용하고자 할때 겪었던 문제와 이를 해결한 기록을 남긴 글이다.

onBackPressedDispatcher 를 어떻게 사용하는지 알고 싶어서 들어오신 분들은 아래 링크를 확인하길 바란다.

[onBackPressed() Deprecated 의 대안 - onBackPressedDispatcher 의 사용](https://gliondev.github.io/posts/Instead-of-OnBackPressed()/)

## 문제 상황
회사에서 서비스 중인 앱의 유지보수를 진행하던 중, 시스템 뒤로가기 버튼 처리를 Deprecated 된 `onBackPressed()` 를 사용하고 있는 것을 발견했다.

최초 배포 이후 기능적인 유지보수만 진행해왔지, 코드레벨의 개선은 이루어지지 않고 있었기에, 이 참에 변경하기로 마음먹고 `onBackPressed()` 내에 정의된 동작을 별도의 `onBackPressedCallback()` 으로 이동, Dispatcher 에 추가하려고 했는데

![자동완성 안됨](https://host.ggoggo.duckdns.org/Blog/250406_onBackPressedDispatcher/cause_1.png)

위 사진처럼 자동완성에 `onBackPressedDispatcher` 가 나타나지 않고,

![직접작성 시 안됨](https://host.ggoggo.duckdns.org/Blog/250406_onBackPressedDispatcher/cause_2.png)

직접 작성해도 **Unresolved reference** 라고 나타나며 `onBackPressedDispatcher` 를 참조할 수 없었다.

## 프로젝트 환경
해당 프로젝트는
* **AGP** : 8.0.2 / **Gradle** : 8.0
* **Gradle JDK** : Java 17
* **소스코드 컴파일 자바 버전** : Java 17
* **Kotlin Version** : 1.8.0
* **compileSdk / targetSdk** : 34
* 타사의 BLE 기기 연동을 위해 제공된 SDK 가 모듈로 추가되어있음

다행히, 작년에 유지보수할때 Gradle 버전, AGP, Kotlin 버전 등은 나름 상위버전으로 올려둔 상태였다. 

~~그래서 당연히 될거라고 생각했다.~~

## 원인 찾기
일단, `onBackPressedDispatcher` 는 `ComponentActivity` 에 생성되어있는 객체이다. 상속 구조에 따라 `AppCompatActivity` 를 상속한 내 `Activity` 에서 `onBackPressedDispatcher` 를 가져와 사용할 수 있는 것이 정상적인 동작이다.
```
정상적인 상속 구조는 이렇게 되어있다.
MainActivity (com.example.myproject)
    └─  AppCompatActivity (androidx.appcompat.app)
        └─FragmentActivity (androidx.fragment.app)
            └─ ComponentActivity (androidx.activity) <- 여기에 onBackPressedDispatcher 객체가 생성되어 있다
                └─ ComponentActivity (androidx.core.app)
                    └─ Activity (android.app)
```

작업중인 내 프로젝트 내 AppCompatActivity 의 상속 구조는 어떻게 되어 있을까?
![](https://host.ggoggo.duckdns.org/Blog/250406_onBackPressedDispatcher/find_1.png)

이 이미지에서의 FragmentActivity 가 상속받는 ComponentActivity 가 androidx.core.app 패키지에 속한 ComponentActivity 이다.

구조를 보면
```
MainActivity (com.example.myproject)
    └─  AppCompatActivity (androidx.appcompat.app)
        └─ FragmentActivity (androidx.fragment.app)
            └─ ComponentActivity (androidx.activity)  <- 이 부분을 건너뛰고
                └─ ComponentActivity (androidx.core.app) <- 바로 core.app 의 ComponentActivity 를 상속 받는다.
                    └─ Activity (android.app)
```
처럼 FragmentActivity 가 androidx.activity 의 ComponentActivity 를 상속받는 것이 아닌, androidx.core.app 의 ComponentActivity 를 상속받고 있다.

**이러니 androidx.activity 의 ComponentActivity 에서 생성하는 onBackPressedDispatcher 객체를 참조할 수 없었던 것이다.**
## 해결 과정
해결하기까지의 과정을 순서대로 정리 해보았다. 모든 과정이 도움이 된 것은 아니지만 해결을 위한 실마리를 점차적으로 잡아나갈 수 있었다.

### 1. androidx.activity:activity:x.y.z 를 명시적으로 참조
`androidx.activity` 의 `ComponentActivity` 를 상속받지 못한다는 것은 어쩌면 `androidx.activity` 가 제대로 import 되지 않았을 수 있다고 생각했다.


따라서, 앱 수준의 `build.gradle` 에서 `androidx.activity:activity:x.y.z` 를 명시적으로 참조하면 되지 않을까 라고 생각했다.


프로젝트 환경에 맞는 버전을 찾아 추가한 뒤, 캐시 제거 -> 프로젝트 클린 -> 리빌드를 진행했지만 변하지 않았다.

### 2. 라이브러리 버전 강제
```groovy
allprojects {
    configurations.all {
        resolutionStrategy {
            force '버전을 강제하고 싶은 라이브러리'
            // ex : force 'androidx.appcompat:appcompat:1.6.1'
        }
    }
}
```
위 코드를 `build.gradle(:project)` 에 추가하여 라이브러리의 버전을 강제할 수 있다고 한다.

gradle 은 기본적으로 최신 버전 우선 정책을 따른다고 하지만, 혹시나 충돌이 발생하여 낮은 버전의 라이브러리가 섞여 이런 현상이 발생하지 않았을까? 해서 추가해보았다.

캐시 삭제, 빌드 clean, rebuild 까지 진행해주었고, 이 역시 효과가 없었다.

(ChatGPT 는 이 방법을 여러번 반복하여 알려주었고 어쩌면 효과가 있을지 모르겠으나, 적어도 내 프로젝트 환경에서는 효과가 없었다.)

### 3. 사용하지 않거나 중복된 라이브러리 참조 제거
레거시한 프로젝트 였기에 여러사람의 손을 거쳤고, 현재로써는 사용하지 않거나 중복되어 제거해도 무방한 라이브러리가 참조되어 있었다.

혹시나 이런 라이브러리들에서 충돌이 났을까 싶어 정리 후 다시 캐시 제거부터 리빌드까지 진행하였지만 역시 원인이 아니였다.

### 4. 라이브러리 의존성 확인
문제 해결을 위해 자료를 찾던 도중, 내 프로젝트에 실제로 포함된 라이브러리의 버전을 알 수 있는 명령이 존재한다는 것을 알았다.

`./gradlew:app:dependencies` 명령어를 사용하면 굉장히 많은 라이브러리들의 의존성 트리를 보여주는데, 라이브러리 들 간 충돌이 발생한 경우, **-> 로 표시된 부분이 버전 충돌 후 선택된 버전을 나타낸다고 한다.**

지금 상황에서 의심가는 라이브러리는 `androidx.appcompat`, `androidx.activity`, `androidx.fragment` 3가지 이므로, `findstr` 옵션을 주어 필요한 라이브러리의 버전만 확인해 보았다.
```shell
./gradlew :app:dependencies --configuration debugRuntimeClasspath | findstr "라이브러리명"
(mac 의 경우 findstr 대신 grep)
```
> (`--configuration debugRuntimeClasspath` 는 디버그 빌드에서 앱을 실행할 때 사용되는 모든 라이브러리와 클래스들의 전체 경로 를 나타내준다)
{: .prompt-info }

각각의 결과는 다음과 같았다.

> androidx.appcompat
```
|    +--- androidx.appcompat:appcompat:1.0.0 -> 1.6.1
|    |    +--- androidx.appcompat:appcompat-resources:1.6.1
|    |    |    \--- androidx.appcompat:appcompat:1.6.1 (c)
|    |    \--- androidx.appcompat:appcompat-resources:1.6.1 (c)
+--- androidx.appcompat:appcompat:1.6.1 (*)
|    +--- androidx.appcompat:appcompat:1.1.0 -> 1.6.1 (*)
|    |    +--- androidx.appcompat:appcompat:1.2.0 -> 1.6.1 (*)
|    \--- com.android.support:appcompat-v7:22.2.0 -> androidx.appcompat:appcompat:1.6.1 (*)
     +--- androidx.appcompat:appcompat:1.4.2 -> 1.6.1 (*)
     |    +--- androidx.appcompat:appcompat:1.4.2 -> 1.6.1 (*)
```

> androidx.fragment
```
|    |    +--- androidx.fragment:fragment:1.3.6 -> 1.5.0
|              \--- androidx.fragment:fragment:1.0.0 -> 1.5.0 (*)
|    +--- androidx.fragment:fragment:1.0.0 -> 1.5.0 (*)
|         +--- androidx.fragment:fragment:1.1.0 -> 1.5.0 (*)
+--- androidx.navigation:navigation-fragment:2.5.0
|    +--- androidx.fragment:fragment-ktx:1.5.0
|    |    +--- androidx.fragment:fragment:1.5.0 (*)
+--- androidx.navigation:navigation-fragment-ktx:2.5.0
|    +--- androidx.navigation:navigation-fragment:2.5.0 (*)
|    +--- androidx.fragment:fragment:1.2.0 -> 1.5.0 (*)
|    |    +--- androidx.fragment:fragment:1.0.0 -> 1.5.0 (*)
|    |    |    \--- androidx.fragment:fragment:1.0.0 -> 1.5.0 (*)
|    +--- androidx.fragment:fragment:1.0.0 -> 1.5.0 (*)
|    +--- androidx.fragment:fragment:1.0.0 -> 1.5.0 (*)
```

> androidx.activity
```
|    |    +--- androidx.activity:activity:1.6.0
|    |    |    \--- androidx.activity:activity-ktx:1.6.0 (c)
|    |    |    +--- androidx.activity:activity:1.5.0 -> 1.6.0 (*)
|    |    +--- androidx.activity:activity-ktx:1.5.0 -> 1.6.0
|    |    |    +--- androidx.activity:activity:1.6.0 (*)
|    |    |    \--- androidx.activity:activity:1.6.0 (c)
|    |    +--- androidx.activity:activity-ktx:1.5.0 -> 1.6.0 (*)
```

-> 가 의미하는 바가 라이브러리가 충돌되어 선택된 버전을 나타낸다고 했는데,
- **`appcompat` 은 `1.0.0` 버전을 비롯한 여러 버전이 충돌되어 `1.6.1` 버전이 선택됨**
- **`fragment` 는 `1.0.0` 버전을 비롯한 여러 버전이 충돌되어 `1.5.0` 버전이 선택되었고**
- **`activity` 는 `1.5.0` 에서 충돌되어 `1.6.0` 버전이 선택되었다.**

**어디에선가 `1.0.0` 버전을 가져오고 있다.**

자료를 찾아봤을떄, `ComponentActivity` 는 `androidx.activity` 의 `1.0.0-alpha01` 버전부터 포함된 객체이고, `FragmentActivity` 및 `AppCompatActivity` 의 새로운 기본 클래스로 도입되었다고 한다.


[Jetpack Activity Android Docs](https://developer.android.com/jetpack/androidx/releases/activity?hl=ko#1.0.0-alpha01)

따라서, `Activity` 는 `1.5.0` 버전에서 충돌되어 `1.6.0` 버전이 선택되었지만, 혹여 `1.5.0` 버전과 꼬였어도 원인은 아닐 것이라 판단했다.

파일 구조를 `project` 로 변경하여 **"External Libraries"** 를 확인해봤을때, 왼쪽 사진을 보면
- **`appcompat`** 라이브러리가 **`1.0.0`**, **`1.6.1`** 2개가 추가되어 있다.
- **`activity`** 라이브러리가 **`1.6.0`**, **`1.5.0`** 2개가 추가되어 있다.
- **`fragment`** 라이브러리가 **`1.0.0`**, **`1.5.0`** 2개가 추가되어 있다.

|        ![설명1](https://host.ggoggo.duckdns.org/Blog/250406_onBackPressedDispatcher/wrong_library_tree.png)        | ![설명2](https://host.ggoggo.duckdns.org/Blog/250406_onBackPressedDispatcher/right_library_tree.png) |
| :----------------------------------------------------------------------------------------------------------------: | :--------------------------------------------------------------------------------------------------: |
| **내 프로젝트의 라이브러리 구조(`activity` 라이브러리도 여러개, `fragment` 라이브러리도 여러개가 참조되어 있다.)** |         **정상적으로 `onBackPressedDispatcher` 가 참조되는 다른 프로젝트의 라이브러리 구조**         |

인 것으로 보아,

_**라이브러리 충돌로 `gradle` 은 가장 최신의 버전을 선택했지만, 모종의 이유로 라이브러리가 꼬여 `AppCompatActivity` 의 상속 트리가 꼬였다는 가설이 어느정도 확실시 되었다.**_

### 5. 타사 SDK 모듈 의존성 확인
내 프로젝트에서는 `androidx.appcompat` 을 `1.6.1` 버전으로, `androidx.core` 를 `1.7.0` 버전으로 사용하고 있다.

`androidx.appcompat` 라이브러리가 배포되어있는 [Maven Repository](https://mvnrepository.com/artifact/androidx.appcompat/appcompat/1.6.1) 를 들어가 `androidx.appcompat` 라이브러리에 대해 확인해보면, `appcompat` 라이브러리에서는 `androidx.activity 1.6.0` 버전을 사용하고 있기 때문에, `1.0.0` 을 사용하고자 하는 원인을 찾아야 했다.

프로젝트에서 사용하는 타사 SDK 를 의심하기 시작했다.

해당 SDK 모듈의 `build.gradle` 을 확인해보니

![sdk module library tree](https://host.ggoggo.duckdns.org/Blog/250406_onBackPressedDispatcher/sdk_library_tree.png)

내 프로젝트에서 사용하는 `appcompat` 보다 한참 낮은 구 버전의 라이브러리를 추가하고 있는 것을 확인할 수 있었다.

해당 모듈 내에서 `appcompat` 을 직접적으로 쓰는 부분이 있는지 확인했고, 이미 deprecated 된 `localbroadcasemanager` 를 제외하곤 `appcompat` 을 사용하는 부분이 없었기에, 해당 의존성을 제거한 뒤
```groovy
implementation 'androidx.localbroadcastmanager:localbroadcastmanager:1.1.0'
implementation 'androidx.core:core:1.7.0'
```
필요한 최소한의 라이브러리만 추가하도록 변경해주었다.

또한 다른 SDK 모듈을 확인해보니, 여기에는 androidx migration 초창기에나 쓰던 Support Library v4 를 androidx 구조로 전환한, lagacy 지원을 위한 라이브러리가 추가되어 있었다.

![sdk module library tree2](https://host.ggoggo.duckdns.org/Blog/250406_onBackPressedDispatcher/sdk_library_tree_2.png)

이 모듈 또한 라이브러리 충돌의 우려가 있는 `androidx.legacy:legacy-support-v4:1.0.0` 을 제거 후 필요한 최소한의 라이브러리만 선택적으로 추가해주었다.

그러고 몇번째인지도 기억 안나는 캐시 제거, 빌드 Clean, 프로젝트 빌드를 수행한 결과...

## 결과
**드디어 `AppCompatActivity` 의 상속 트리가 정상적으로 나타났고, `Activity` 단에서 `onBackPressedDispatcher` 또한 참조되어 사용이 가능해졌다!**

4번 과정에서 확인한 External Libraries 를 확인해보니, `activity` 는 여전히 `1.5.0`, `1.6.0` 이 존재하지만,
`appcompat` 은 `1.6.1` 1개만 존재하고, `fragment` 또한 `1.5.0` 1개만 존재하는 것을 확인할 수 있다.

![complete library tree](https://host.ggoggo.duckdns.org/Blog/250406_onBackPressedDispatcher/complete_tree.png)

**외부 모듈을 쓸 때는, 해당 모듈에서 참조중인 라이브러리를 확인하고 메인 모듈(:app) 에 중복되는 라이브러리가 존재한다면, 어떠한 이유로던 영향을 끼칠 수 있다는 점을 인지하고 유의해야 할 것이다.**