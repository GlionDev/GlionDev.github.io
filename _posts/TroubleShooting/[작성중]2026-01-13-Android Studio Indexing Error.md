---
title: "AGP 8.5  Gradle 8.7 환경에서 Version Catalog(libs) 사용 시 Android Studio 인덱싱 오류 사례"
date: 2026-01-13 15:31:44 +0900
categories: [TroubleShooting]
tags: [TroubleShooting, AGP, Gradle, libs.version.toml]
---

## 문제 상황
- AGP : 8.5
- Build Gradle : 8.7
- IDE : Android Studio Otter 2 Feature Drop 2025.2.2

build.gradle.kts(:app) 에서 libs.version.toml 을 사용했을 경우, 아래 이미지처럼 모든 코드가 `Unresolved Reference` 로 나타난다.
```
이미지 영역
![]()
```

build.gradle.kts 에 새로운 의존성을 추가하거나, 변경할때 자동완성이 되지 않고 뭔가 오류가 난 것 같은 느낌을 준다.

빌드는 정상적으로 이루어지는 것으로 보아, IDE 의 인덱싱 오류로 판단.

> 위와 같은 현상이 나타날 경우, 최우선적으로 `.gradle` 과 `.idea` 폴더를 제거한 뒤 Invalidate Caches 를 수행 후 프로젝트를 다시 열어보는것을 추천한다. 대부분의 경우 이 방법으로 해결이 된다.
{: .prompt-tip }

## 재연
프로젝트를 하나 만들어서, AGP 와 Gradle 을 위 버전과 동일하게 두고, version catalog 를 사용할 경우 마찬가지로 인덱싱 오류 발생함.

## 해결책
### 1. buildSrc 사용
기존 방식인 `id(...)` 를 사용하게 되면 라이브러리 별 버전을 한눈에 파악할 수 없어 버전 관리가 어렵다는 단점이 존재한다.

따라서 `buildSrc` 로 의존성을 주입하는 방법을 사용한다.

1. 프로젝트 루트 경로에 `buildSrc` 폴더를 생성한다.
    > 모듈을 만드는 것이 아님에 유의한다.
    {: .prompt-warning }
2. `buildSrc` 하위에 `build.gradle.kts` 를 생성 후 아래 내용을 넣는다.
    ```gradle
    plugins {
        `kotlin-dsl`
    }

    repository {
        google()
        mavenCentral()
    }
    ```
3. `buildSrc` 하위에 `src/main/kotlin` 폴더 구조를 만든다.
4. kotlin 하위에 object 로 목적에 맞게 생성한다. 필자는 아래와 같이 사용하였다.  
   1. **`AppConfig.kt`** : `compileSdk`, `minSdk`, `targetSdk`, `appVersionCode`, `appVersionName` 을 지정, 관리
   2. **`Version.kt`** : 플러그인 및 라이브러리 버전 관리
   3. **`Plugin.kt`** : 플러그인 관리
   4. **`Library.kt`** : 라이브러리 경로 및 버전 명시, 관리
5. gradle sync 한 뒤 build.gradle.kts 에 정의한 설정을 사용한다.

예시

AppConfig.kt
```kotlin
object AppConfig {
    const val COMPILE_SDK: Int = 34
    const val MIN_SDK: Int = 29
    const val TARGET_SDK: Int = 34
    const val APP_VERSION_CODE: Int = 1
    const val APP_VERSION: String = "0.0.1"
}
```
Versions.kt
```kotlin
object Versions {
    const val AGP = "8.5.0"
    const val KOTLIN = "1.9.0"
    const val CORE_KTX = "1.10.1"
    const val JUNIT = "4.13.2"
    const val JUNIT_VERSION = "1.1.5"
    const val ESPRESSO_CORE = "3.5.1"
    const val APPCOMPAT = "1.6.1"
    const val MATERIAL = "1.10.0"
    const val ACTIVITY = "1.8.0"
}
```
Plugin.kt
```kotlin
object Plugin {
    const val ANDROID_APPLICATION = "com.android.application"
    const val JETBRAINS_KOTLIN_ANDROID = "org.jetbrains.kotlin.android"
}
```
Library.kt
```kotlin
object Library {
    const val ANDROIDX_CORE_KTX = "androidx.core:core-ktx:${Versions.CORE_KTX}"
    const val ANDROIDX_APPCOMPAT = "androidx.appcompat:appcompat:${Versions.APPCOMPAT}"
    const val MATERIAL = "com.google.android.material:material:${Versions.MATERIAL}"
    const val ANDROIDX_ACTIVITY = "androidx.activity:activity:${Versions.ACTIVITY}"
    const val ANDROIDX_CONSTRAINT_LAYOUT = "androidx.constraintlayout:constraintlayout:${Versions.CONSTRAINT_LAYOUT}"

    // Test
    const val JUNIT = "junit:junit:${Versions.JUNIT}"
    const val ANDROIDX_JUNIT = "androidx.test.ext:junit:${Versions.JUNIT_VERSION}"
    const val ANDROIDX_ESPRESSO_CORE = "androidx.test.espresso:espresso-core:${Versions.ESPRESSO_CORE}"
}
```

build.gradle.kts(:project)
```kotlin
plugins {
    id(Plugin.ANDROID_APPLICATION) version Versions.AGP apply false
    id(Plugin.JETBRAINS_KOTLIN_ANDROID) version Versions.KOTLIN apply false
}
```

build.gradle.kts(:app)
```kotlin
plugins {
    id(Plugin.ANDROID_APPLICATION)
    id(Plugin.JETBRAINS_KOTLIN_ANDROID)
}
...
android {
    namespace = "com.example.sample"
    compileSdk = AppConfig.COMPILE_SDK

    defaultConfig {
        applicationId = "com.example.sample"
        minSdk = AppConfig.MIN_SDK
        targetSdk = AppConfig.TARGET_SDK
        versionCode = AppConfig.APP_VERSION_CODE
        versionName = AppConfig.APP_VERSION
}
...
dependencies {
    implementation(Library.ANDROIDX_CORE_KTX)
    implementation(Library.ANDROIDX_APPCOMPAT)
    implementation(Library.MATERIAL)
    implementation(Library.ANDROIDX_ACTIVITY)
    implementation(Library.ANDROIDX_CONSTRAINT_LAYOUT)
    ...
    testImplementation(Library.JUNIT)
    testImplementation(Library.ANDROIDX_JUNIT)
    testImplementation(Library.ANDROIDX_ESPRESSO_CORE)
}
```

### 2. AGP 와 Gradle 버전 업데이트
