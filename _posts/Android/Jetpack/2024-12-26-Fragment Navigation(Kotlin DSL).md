---
title: 'Fragment Navigation - 2 (Using Kotlin DSL)'
date: 2024-12-26 11:30:00 +0900
categories: [Android, Jetpack]
tags: [jetpack, fragment, navigation, DSL]
---

이전 포스팅인 **[Android] SAA 와 Fragment Navigation - 1 (XML Based Graph)** 에서 이어진다.

> **이전 글 보러가기**
> [[Android] SAA 와 Fragment Navigation - 1 (XML Based Graph)](https://gliondev.github.io/posts/Fragment-Navigation-XML-Based-Graph/)
{: .prompt-info }

## 개요
Android Docs 의 Navigation 파트의 `탐색 그래프 설계 - 개요` 를 보면 프로그래매틱 방식을 소개하고 있다.

여기서는 이전 포스팅에서 graph 방식으로 구현한 Navigation 을 DSL 방식으로 재구현 해본다.

## DSL 이란?
> **DSL (Domain Specific Language)**
> 특정 도메인에 대한 목적으로 만들어진 언어를 의미한다. 상용구 코드를 최소화 하기 위해 명령형 코드 대신 선언적 코드 형식을 따른다.
{: .prompt-info }

이전 `build.gradle(앱 수준)` 이나 `build.gradle(프로젝트 수준)` 은 groovy 언어로 작성되어 있었는데 이는 Gradle 스크립팅을 하는 것을 목적으로 하는 DSL 의 하나의 예시이다.

지금은 Groovy DSL 에서 Kotlin DSL 로 변경되었다. 이에 대한 자세한 내용은 [Migrating Groovy DSL to Kotlin DSL](https://charlezz.com/?p=45140) 을 참고하자.

Kotlin DSL 은 특정 목적(Navigation) 을 수행할 수 있는 방법을 제공하고 있다.

## 환경 설정
프래그먼트와 함께 Kotlin DSL 을 사용하기 위해서는 `build.gradle(앱 수준)` 에 아래를 추가한다.

```kotlin
dependencies {
    val nav_version = "2.8.4" // 구글 공식문서에 작성되어있는 버전

    api("androidx.navigation:navigation-fragment-ktx:$nav_version")
}
```

또한, XML 기반의 네비게이션 그래프는 **빌드 과정에서** 파싱되어 그래프에 정의된 각 ID 속성에 대해 숫자 상수가 생성되게 된다.

그러나 이렇게 생성된 ID 는 아래에서 다룰 프로그래매틱 방식은 런타임에 네비게이션 그래프를 생성할 때 사용할 수 없다고 한다.

따라서 Navigation DSL 에서는 ID 대신 직렬화 가능한 고유한 타입을 사용해야 한다.

직렬화를 위해 아래의 내용도 추가해준다.

```kotlin
plugins {
    // ...
    kotlin("plugin.serialization") version "2.0.21" // 직렬화 플러그인 추가
}

// ...

dependencies {
    // ...
    // JSON serialization library, works with the Kotlin serialization plugin
    implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.7.3")
    // ...
}
```

## Navigation 그래프 만들기
프로그래매틱 방식으로 그래프를 만들때는, MainActivity 에서 FragmentContainerView 를 선언할때 `app:navGraph` 선언을 제외하고 선언한다.

xml 방식으로 graph를 생성하지 않으니 `app:navGraph` 를 선언하지 않는 것이다.

그걸 제외하고는 동일하게 Fragment 를 띄울 Host 를 MainActivity 의 레이아웃에 지정해준다.

```xml
<androidx.fragment.app.FragmentContainerView
    android:id="@+id/nav_host_dsl"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:name="androidx.navigation.fragment.NavHostFragment"
    app:defaultNavHost="true"/>
```

이제 Host 에서 사용할 graph 를 NavGraphBuilder DSL 을 사용하여 생성해주어야 한다.

MainActivity 에서 NavController 를 가져온 뒤, `createGraph()` 를 통해 그래프를 생성한다.(NavController 를 가져오는 방법은 이전 포스팅을 참고한다)

위의 환경설정 하면서, Navigation DSL 에서는 ID 대신 직렬화 가능한 고유한 타입을 사용한다고 했다.

구글 공식문서에서는

```kotlin
@Serializable data object Home
@Serializable data class Plant(val id: String)
```

와 같이 선언한 뒤, Fragment label에 string resource 에 선언한 내용을 넣어주고 있다.

인자를 전달하고자 할때에는 타입에 인자로서 전달하고자 하는 데이터를 넣어줄 수 있다.

```kotlin
navController.graph = navController.createGraph(
    startDestination = Home
) {
    fragment<HomeFragment, Home> {
        label = resources.getString(R.string.home_title) // strings.xml 에 각 Fragment 에 대한 label 을 따로 선언해주어야 한다.
    }
    fragment<PlantDetailFragment, PlantDetail> {
        label = resources.getString(R.string.plant_detail_title)
    }
}
```

이 예시는 `fragment()` DSL 빌더 함수를 이용하여 두개의 Fragment 에 대한 destination(목적지) 을 정의하고 있다. 이 fragment() 함수는 2개의 인자가 필요하다.

`fragment()` 함수를 살펴보면 아래와 같이 되어있다.

```kotlin
/**
 * Construct a new [FragmentNavigator.Destination]
 *
 * @param T the destination's unique route from a [KClass]
 * @param typeMap map of destination arguments' kotlin type [KType] to its respective custom
 * [NavType]. May be empty if [T] does not use custom NavTypes.
 */
public inline fun <reified F : Fragment, reified T : Any> NavGraphBuilder.fragment(
    typeMap: Map<KType, @JvmSuppressWildcards NavType<*>> = emptyMap(),
): Unit = fragment<F, T>(typeMap) {}

/**
 * Construct a new [FragmentNavigator.Destination]
 *
 * @param T the destination's unique route from a [KClass]
 * @param typeMap map of destination arguments' kotlin type [KType] to its respective custom
 * [NavType]. May be empty if [T] does not use custom NavTypes.
 * @param builder the builder used to construct the fragment destination
 */
public inline fun <reified F : Fragment, reified T : Any> NavGraphBuilder.fragment(
    typeMap: Map<KType, @JvmSuppressWildcards NavType<*>> = emptyMap(),
    builder: FragmentNavigatorDestinationBuilder.() -> Unit
): Unit =
    destination(
        FragmentNavigatorDestinationBuilder(
                provider[FragmentNavigator::class],
                T::class,
                typeMap,
                F::class,
            )
            .apply(builder)
    )
```

첫번째 인자 F는 destination 에 대한 화면에 표시해줄 Fragment 가 들어간다. graph 방식에서 각 fragment 에 대한 `android:name="..."` 에 해당한다.

두번째인자 T는 경로이다. graph 방식에서 id 에 해당하는 부분으로서, 반드시 직렬화 가능한 타입이여야 한다.(추가한 직렬화 플러그인을 사용한다 - `@Serializable`)

또한 `fragment()` 함수는 label 이나 전달하고자 하는 데이터, 딥링크 등의 추가적인 구성을 람다함수를 통해 추가할 수 있도록 되어있다.

## Route 변형
구글 공식문서에서 제공한 Route 생성 코드를 보면, Fragment와 매핑될 Route 를 만들고, label 에 대한 값을 strings.xml 에 정의하여 사용하고 있는 것을 확인할 수 있다.

이 방식이 번거롭다고 느껴져, Route 내에 label 을 정의하고 가져올 수 있는 방식으로 변경해보자. Sealed Class 를 사용할 것이다.

이전 포스팅에서 사용했던 화면 구조를 따라가게 되면 아래와 같이 정의할 수 있다.

```kotlin
@Serializable
sealed class AppRoute(val label: String) {
    @Serializable data object Home : AppRoute(label = "home")
    @Serializable data class Option1(val sendValue: Int) : AppRoute(Companion.label) {
        companion object {
            const val label = "option1"
        }
    }
    @Serializable data object Option2 : AppRoute(label = "option2")
    @Serializable data object Option2_1 : AppRoute(label = "option2_1")
    @Serializable data class Result(val resultData: DslResultData) : AppRoute(Companion.label) {
        companion object {
            const val label = "result"
        }
    }
    @Serializable data object Find : AppRoute(label = "find")
}
```

전체 Sealed Class 에 인자로 label 을 두고, 각 Route 마다 label 을 정의해준다. 그 뒤 `fragment()` 함수에서 label 을 아래와 같이 지정할 수 있다.

(sendValue 나 resultData 는 우선 무시한다. 아래에서 navigate 시 인자전달에 대해 다룰때 언급할 예정이다)

```kotlin
navController.graph = navController.createGraph(
    startDestination = AppRoute.Home
) {
    fragment<FragmentDslHome, AppRoute.Home> {
        label = AppRoute.Home.label
    }
    fragment<FragmentDslOption1, AppRoute.Option1> {
        label = AppRoute.Option1.label
    }
    fragment<FragmentDslOption2, AppRoute.Option2> {
        label = AppRoute.Option2.label
    }
    fragment<FragmentDslOption2_1, AppRoute.Option2_1> {
        label = AppRoute.Option2_1.label
    }
    fragment<FragmentDslResult, AppRoute.Result>(
        typeMap = mapOf(typeOf<DslResultData>() to ResultDataParametersType)
    ) {
        label = AppRoute.Result.label
    }
    fragment<FragmentDslFind, AppRoute.Find> {
        label = AppRoute.Find.label
    }
}
```

## 생성한 Graph 를 통한 화면 이동

이런 식으로 graph 를 생성했다면, 이동하고자 하는 Fragment 에서 `findNavController.navigate(Route)` 를 통해 화면 이동이 가능하다.

`fragment()` 를 통해 특정 Fragment 와 Route로 Destination 을 생성하였고, navigate 시 route 를 인자로 넣어주어 route 에 해당하는 destination 을 찾아 특정 Fragment 를 화면에 띄워주는 방식이다.

```kotlin
// navigate from PlantDetail to Home
findNavController().navigate(route = Home)
```

Sealed Class 로 생성한 Route 를 이용한다면

```kotlin
findNavController().navigate(route = AppRoute.Home)
```
로 사용할 수 있겠다.

## Destination 으로 인자 전달

### 원시타입(Int, Float, Double, Long, Boolean, String...)
원시 타입의 전달은 매우 간단하다.

예를 들어, FragmentDslHome 에서 FragmentDslOption1 로 이동할때, Int 값을 전달한다고 가정해보자.

값을 받을 Fragment 의 Route 에 매개변수에 전달할 값을 넣어준다.

```kotlin
@Serializable data class Option1(val sendValue: Int /* 전달할 Int 값 */) : AppRoute(Companion.label) {
    companion object {
        const val label = "option1"
    }
}
```

그 뒤, FragmentDslHome 에서 navigate 시 전달할 인자를 넣어준다.

```kotlin
findNavController().navigate(route = AppRoute.Option1(sendValue = 100))
```

이제, 값을 받을 FragmentDslOption1 에서 bundle 로 전달된 값을 받을때처럼 해주면 전달된 값을 확인할 수 있다.

```kotlin
class FragmentDslOption1 : Fragment() {
    lateinit var binding: FragmentDslOption1Binding
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val getValue = arguments?.getInt("sendValue") // key 값으로 변수명이 들어간다.
        Log.d("glion", "원시타입 데이터 전달 :: $getValue")
    }
    // ...
}
```

### 커스텀 데이터(Data Class)
실질적으로 원시타입을 전달하는것 보다 특정 데이터 클래스를 전달할 경우가 더 많을 것이다.

FragmentDslResult 에서 DslResultData 를 받는다고 가정할때, 값을 받을 Fragment 의 Route 에 전달할 값을 넣어주는것은 동일하다.

```kotlin
@Serializable data class Result(val resultData: DslResultData) : AppRoute(Companion.label) {
    companion object {
        const val label = "result"
    }
}
```

DslResultData 는 직렬화 가능한 타입이여야 한다.

```kotlin
@Serializable
@Parcelize
data class DslResultData(
    val title: String,
    val value: Int
) : Parcelable
```

여기서 중요한 과정이 하나 더 존재하는데, 바로 전달하고자 하는 데이터 타입을 NavType 으로 변경해주어야 한다.

```kotlin
val ResultDataParametersType = object : NavType<DslResultData>(
    isNullableAllowed = false
) {
    override fun get(bundle: Bundle, key: String): DslResultData? {
        return bundle.getParcelable(key) as DslResultData?
    }

    override fun parseValue(value: String): DslResultData {
        return Json.decodeFromString<DslResultData>(value)
    }

    override fun put(bundle: Bundle, key: String, value: DslResultData) {
        bundle.putParcelable(key, value)
    }

    override fun serializeAsValue(value: DslResultData): String {
        return Json.encodeToString(DslResultData.serializer(), value)
    }
}
```

그 뒤 createGraph 시 `fragment()` 함수를 통해 destination 을 정의할때 전달하고자 하는 데이터를 인자 treeMap 에 넣어준다.

```kotlin
fragment<FragmentDslResult, AppRoute.Result>(
    typeMap = mapOf(typeOf<DslResultData>() to ResultDataParametersType)
) {
    label = AppRoute.Result.label
}
```

DslResultData 의 각 property 를 map 형태로 저장하여 전달하게 되는데, 전달을 위해 DslResultData(일반 Data class) 를 `NavType<DslResultData>` 로 변환한다.

마지막으로, 전달할 인자를 navigate 시 넣어준다.

```kotlin
val resultData = DslResultData("Option1 에서 옴", Random.nextInt(1, 100))
findNavController().navigate(AppRoute.Result(resultData))
```

값을 받는 곳에서도 조금 다른데, navController 에서 Route 에 있는 resultData 를 가져오는 형태로 전달된 DslResultData 를 받아 사용이 가능하다.

```kotlin
private val args: DslResultData by lazy {
    findNavController().getBackStackEntry<AppRoute.Result>().toRoute<AppRoute.Result>().resultData
}
```

이 과정은 오히려 Graph 방식보다 번거로운 느낌을 받을 수 있는데, 화면이 많아지고 전달해야 할 데이터가 많아질 수록 전달하고자 하는 Object 를 NavType 으로 변경해주는 작업이 추가적으로 더 필요하기 때문이다.

## 애니메이션 추가
Navigation DSL 에서의 Fragment 전환 간 Animation 처리는 Graph 방식보다 불편하다.

우선 Graph 방식에서 Animation 처리는 graph action 내에서 `enterAnim`, `exitAnim`, `popEnterAnim`, `popExitAnim` 속성에 애니메이션만 넣어주면 됐었다.

이런식으로 말이다.

```xml
<action
    android:id="@+id/action_fragmentHome_to_fragmentOption1"
    app:destination="@id/fragmentOption1"
    app:enterAnim="@anim/slide_in"
    app:exitAnim="@anim/nav_default_exit_anim"
    app:popEnterAnim="@anim/nav_default_pop_enter_anim"
    app:popExitAnim="@anim/slide_out" />
```

그러나 DSL 에서는 navigate 시 NavOptions 객체를 생성하여 navigate 시 함께 전달해주어야 한다.

위의 애니메이션 처리를 NavOption 객체를 만들어 navigate 시 전달하게 되면, 아래와 같이 사용할 수 있다.

```kotlin
val naviOptions = NavOptions.Builder()
    .setEnterAnim(R.anim.slide_in)
    .setExitAnim(androidx.navigation.ui.R.anim.nav_default_exit_anim)
    .setPopEnterAnim(androidx.navigation.ui.R.anim.nav_default_pop_enter_anim)
    .setPopExitAnim(R.anim.slide_out)
    .build()
    
findNavController().navigate(AppRoute.Home, navOptions) // navigate 시 navOptions 객체를 함께 전달해준다.
```

그러나 이렇게 하게 되면 모든 navigate 하는 부분에 navOptions 객체를 생성하여 추가해줘야 하는 불편함이 존재한다.

navigate 시 애니메이션이 매번 다르다면 어쩔수 없지만, **전체적으로 동일할 경우 보일러 플레이트 코드가 늘어나, 생산성이 떨어질 수 있다.**

따라서, 샘플 프로젝트에서는 Kotlin 의 확장함수 기능을 사용하여 아래와 같이 적용하였다.

```kotlin
fun NavController.navigateWithAnim(
    route: AppRoute,
    isInclusive: Boolean = false,
    @AnimRes enterAnim: Int = R.anim.slide_in,
    @AnimRes exitAnim: Int = androidx.navigation.ui.R.anim.nav_default_exit_anim,
    @AnimRes popEnterAnim: Int = androidx.navigation.ui.R.anim.nav_default_pop_enter_anim,
    @AnimRes popExitAnim: Int = R.anim.slide_out
) {
    val naviOptions = NavOptions.Builder()
        .setEnterAnim(enterAnim)
        .setExitAnim(exitAnim)
        .setPopEnterAnim(popEnterAnim)
        .setPopExitAnim(popExitAnim)
    if(isInclusive) {
        naviOptions.setPopUpTo(route, true)
    }

    navigate(route, naviOptions.build())
}
```

`isInclusive` 파라미터는 graph 에서

```xml
app:popUpTo="@id/fragmentHome"
app:popUpToInclusive="true"
```

부분을 구현한 것이다. `setPopUpTo` 함수를 확인해보면

```kotlin
/**
 * Pop up to a given destination before navigating. This pops all non-matching destinations
 * from the back stack until this destination is found.
 ...
 */
@JvmOverloads
@Suppress("MissingGetterMatchingBuilder")
@OptIn(InternalSerializationApi::class)
public fun <T : Any> setPopUpTo(
    route: T,
    inclusive: Boolean,
    saveState: Boolean = false
): Builder {
    popUpToRouteObject = route
    setPopUpTo(route::class.serializer().generateHashCode(), inclusive, saveState)
    return this
}
```
이렇게 되어 있고 inclusive 에 true 를 주어 navigate 하면서 백스택에 남아있는 화면을 포함하여 pop 할것인지에 대한 옵션을 추가해 줄 수 있다.

## 정리
이렇게 graph 로 구현한 navigation 을 Kotlin DSL 을 사용하여 동일하게 구현해 보았다.

지금은 compose 기반 UI 로 대부분 변경되어 Fragment 에서만 쓰이던 XML 기반의 graph 는 자연스럽게 사라지고 있는 추세이다.

추후 Compose 방식으로 앱을 제작할때를 대비하여 그와 비슷한 DSL 방식으로 구현해보는 연습을 해보면 분명 도움이 될 것이라 생각한다.

## 샘플 프로젝트
해당 글을 작성하며 사용된 샘플 프로젝트이다.

`graph_dsl_on_activity` 브랜치의 NavigationFragmentSample 폴더 내 `dsl` 패키지에서 위의 내용을 확인 할 수 있다.

[샘플 프로젝트 바로가기](https://github.com/GlionDev/AndroidStudy/tree/graph_dsl_on_activity/NavigationFragmentSample)

## 참고자료
- [Android Docs Navigation](https://developer.android.com/guide/navigation?hl=ko)
- [Migrating Groovy DSL to Kotlin DSL](https://charlezz.com/?p=45140)