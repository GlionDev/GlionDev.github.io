---
title: "TextSize \"dp\" 로 고정해 둔 값이 풀리는 경우"
date: 2023-11-02 17:39:03 +0900
categories: [TroubleShooting]
tags: [Android, TroubleShooting, Dp, TestSize, Xml, Full Screen]
---

기기마다 다르게 적용되는 반응형 UI를 충분히 구현하지 못해 차선책으로 TextSize를 dp로 고정하는 방식을 사용하여 프로젝트를 진행하던 중, 동영상을 재생하는 Activity로 이동했다가 동영상이 종료되어 다시 MainActivity로 이동하였을 때 고정해둔 값이 풀리는 현상을 발견하여 이에 대해 정리한다.
예시로 제시하고 있는 코드는 실제 프로젝트와 동일한 흐름으로 동작하도록 만든 샘플 프로젝트이다.

## 조건
프로젝트에서 사용한 조건은 아래와 같다.

1. MainActivity 는 BaseActivity 를 상속하고, 화면 크기와 상관없이 View 의 크기를 동일하게 고정하기 위해, BaseActivity 의 onCreate 에서 DPI 값을 고정시켜주었다.
    ```kotlin
    abstract class BaseActivity : AppCompatActivity() {
        val TAG = "shhan"
        override fun onCreate(savedInstanceState: Bundle?) {
            super.onCreate(savedInstanceState)

            val configuration: Configuration = resources.configuration
            // MEMO : 앱의 폰트 스케일을 1로 고정
            configuration.fontScale = 1.toFloat() //0.85 small size, 1 normal size, 1,15 big etc

            val metrics = DisplayMetrics()
            // MEMO : 디스플레이의 디스플레이 측정값을 가져옴
            windowManager.defaultDisplay.getMetrics(metrics)
            // MEMO : 앱의 DPI를 x축 인치당 도트 수로 설정함
            configuration.densityDpi = resources.displayMetrics.xdpi.toInt()
            // MEMO : 위에서 설정한 구성을 앱 전체에 적용시킴
            baseContext.resources.updateConfiguration(configuration, metrics)
        }
    }
    ```
2. MainActivity 위에 3개의 `Fragment` A, B, C 가 있고 메인으로 B Fragment 가 제일 처음 나타난다. B 에는 동영상을 재생하는 버튼이 있고, 누르면 동영상이 재생되는 Activity 가 시작된다.
3. 동영상이 재생되는 VideoActivity 는 FullScreen Theme 가 적용된다. 하단바를 없애고, 상단 엣지영역까지 전부 렌더링되어 전체화면으로 동영상이 재생된다.
4. MainActivity 와 VideoActivity 는 Manifest에서 configChanges 속성으로 화면 회전시에 Activity를 다시 시작하지 않는다. 또한 MainActivity에서 VideoAcitivity 를 시작할때, finish()를 호출하지 않아 MainActivity는 종료되지 않고 백그라운드 상태가 된다.
5. 프로젝트에 적용된 dependencies 는 다음과 같다.
    ```kotlin
    dependencies {
        implementation("androidx.core:core-ktx:1.7.0")
        implementation("androidx.appcompat:appcompat:1.4.1")
        implementation("com.google.android.material:material:1.5.0")
        implementation("androidx.constraintlayout:constraintlayout:2.1.3")
        
        // exoplayer
        implementation("com.google.android.exoplayer:exoplayer:2.16.1")
        implementation("androidx.legacy:legacy-support-v4:1.0.0")
    }
    ```
6. Project Structure 은 아래와 같다.
    - Android Gradle Plugin : 7.1.1
    - Gradle Version : 7.2
    - Gradle JDK : jbr-17

## 에러 확인
<video 
    src="https://host.ggoggo.duckdns.org/Blog/231102_dp_fixed/check_error.mp4" 
    controls 
    controlsList="nodownload"
    oncontextmenu="return false;"
    style="max-width: 30%; display: block; margin: 0 auto;">
</video>
보이는 것처럼 시스템 화면크기를 제일 크게 해도 BaseActivity 에 작성한 코드의 영향으로 정해진 사이즈로 View 가 나타났지만, 동영상을 재생한 뒤 종료하면 갑자기 커지는 모습을 확인할 수 있다.

## 원인 파악
dp로 고정했음에도 풀리는 원인에 대해 찾던 중 BaseActivity에서 configuration 의 dpi을 고정했던 것을 기억하고,
Activity와 Fragment 의 각 생명주기에서 configuration의 dpi값을 확인해보았다.  
아래 명령어는 애플리케이션의 영향을 주는 장치 구성 정보 중에서 DPI 값만 출력하도록 한 것이다.

```kotlin
Log.d(TAG, "${context.configuration.densityDpi}"
// context 는 각 Activity, Fragment가 가지고 있는 context를 의미함
```

샘플프로젝트에서 찍어본 로그는 아래와 같았다. 실제 프로젝트 또한 로그를 찍었을때 동일하게 나타났다.  
(실제 프로젝트에서는 통신에 걸리는 시간과 동영상 파일을 다운로드 받는 시간이 있었기에, 샘플 프로젝트에서는
스레드로 비슷한 타이밍을 구현해냈다)

![verticalArrangement 종류](https://host.ggoggo.duckdns.org/Blog/231102_dp_fixed/dpi_log.png)  
playbackState 가 STATE_BUFFERING (2)일때 DPI 와&nbsp; STATE_READY (3) 일때의 DPI 값이 다르다.

이로부터 2가지를 확인할 수 있었다.  
1. **MainActivity가 onStop이 되었을때 화면을 구성하는 DPI 값이 변경되었다는 점.** 
   - 이부분때문에 동영상 재생이 끝나고 MainActivity 돌아갔을때 textSize가 커져있었던 원인이 될 것이고
2. **VideoActivity 에서 playbackState(Exoplayer의 onPlaybackStateChanged 함수는 playback state가 변화될때마다 호출됨) 값이 2(STATE_BUFFERING) 에서 3(STATE_READY) 이 될때 DPI 값이 변경되었다는 점.** 
    - 따라서 VideoActivity에서 다른 Fragment 나 Dialog 등이 나타날 때에도 의도했던 사이즈와 다르게 나올 수 있다는 점이다.

DPI가 변하면 화면에 그리는 사이즈가 달라진다.  
우리가 Layout을 만들때 dp, sp 와 같은 단위로 크기를 정하지만, 내부적으로 px로 변환되어 그려지게 되는데, `px = dp x (dpi/160)` 의 공식에 따라 변환된다.  
1dp로 지정했다고 할지라도 dpi가 커지면 실제 화면에 그려지는 px 값은 커지게 된다.

기존 480 DPI 였을때 10dp로 지정한 textSize는 30px이 되어 화면에 그려지는데, 어떠한 원인으로 인해 DPI가 800으로 증가되었다면 50px이 되어 화면에 그려지게 된다.

## 해결 방안
두가지 해결방안이 존재한다.

1. BaseContext 의 DPI 값을 고정시켜주는 부분 코드를 변경하는 것  
    BaseActivitiy 의 onCreate에 작성헀던 코드의 마지막줄 `updateConfiguration` 은 `deprecated`된 함수이다.
    이를 변경시켜주면 해결이 되었다.  

    먼저 DPI 값을 고정한 context를 반환하는 object 를 하나 만든다.
    ```kotlin
    object FixConfiguration {
        fun wrap(context: Context): Context{
            val res = context.resources
            val configuration: Configuration = res.configuration
            // MEMO : 앱의 폰트 스케일을 1로 고정
            configuration.fontScale = 1.toFloat() //0.85 small size, 1 normal size, 1,15 big etc
            // MEMO : 앱의 DPI를 x축 인치당 도트 수로 설정함
            configuration.densityDpi = res.displayMetrics.xdpi.toInt()
            // MEMO : 위에서 설정한 구성을 적용한 context 를 리턴
            return context.createConfigurationContext(configuration)
        }
    }
    ```
    그리고 BaseActivity 에서 attachBasecontext 를 오버라이드 하여, 생성된 context 를 생성자의 인자로 넣어준다.  

    기존 onCreate 의 DPI 고정하는 부분은 제거해준다.
    ```
    override fun attachBaseContext(newBase: Context?) {
        super.attachBaseContext(FixConfiguration.wrap(newBase!!))
    }
    ```
    실행해보면, 로그에 아래와 같이 찍히는걸 확인 할 수 있다.
    ![첫번째 해결방법](https://host.ggoggo.duckdns.org/Blog/231102_dp_fixed/fix_solve_1.png)  

    원래였다면 `VideoActivity onConfigurationChanged` 부분에 화면이 회전했을떄 DPI 값이 변화했어야 하지만, 앱 전체에 영향을 주는 `Context` 는 애플리케이션이 실행될 때 `FixConfiguration` 으로 고정한 값이 적용되어 있기 때문에 DPI 가 변하지 않는다.

    오버라이드 함수인 attachBaseContext 는 아래 링크에서 자세히 설명하고 있다.  
    간략하게 설명하면, `Context` 를 직접 상속한 `ContextWrapper` 는 `Activity`, `Service`, `Application` 의 슈퍼클래스이고, 이들은 `attachBaseContext` 를 사용하여 `Context` 를 설정한다.  

    우리는 `attachBaseContext` 의 인자로 `FixConfiguration` 으로 설정한 Context 를 넣어주기 때문에 고정된 DPI 를 가질 수 있었다.  

    [attachBaseContext 에 대한 자세한 설명 - 출처 : Velog 블로그 wonseok.log](https://velog.io/@ows3090/Android-Context-ContextWrapper-ContextImpl-%EC%9D%B4%ED%95%B4)

2. appcompat 과 material 버전 수정  
    build.gradle(:app) 의 `dependencies` 에 기본적으로 추가되어있는
    ```kotlin
    implementation("androidx.appcompat:appcompat:1.4.1")
    implementation("com.google.android.material:material:1.5.0")
    ```

    이 두가지의 버전을 각각 `1.4.1 -> 1.6.1`, `1.5.0 -> 1.9.0` 으로 올려주니 `BaseActivity` 를 수정하지 않고, Deprecated 된 방법을 사용해도 DPI 가 변하지 않는 것을 확인 할 수 있었다.

    공식문서 내 **Version 1.6.0-alpha04** 를 보면  
    [AppCompat 공식문서](https://developer.android.com/jetpack/androidx/releases/appcompat#version_142_3)  
    > BugFixs
    > * Avoid managed configuration when config changes outside of attachBaseConfig

    라고 되어있는데, `1.6.0` 버전 이전에는 `attachBaseConfig` 외부에서 `configuration` 의 변화를 막지 않았기 때문에 위와 같은 현상이 발생할 수 있었음을 알 수 있다.

    > 이 방법은 샘플 프로젝트에서는 통하지 않았지만, 실제 문제가 생긴 프로젝트에서는 적용됐던 방법이다.
    {: .prompt-warning }