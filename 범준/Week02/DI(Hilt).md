## DI란?

DI는 객체가 자신의 의존성을 직접 생성하지 않고 외부에서 주입받도록 해서 결합도를 낮추고 테스트를 쉽게 하고 교체 가능성을 높이는 방식이다. Hilt는 Android에서 이 DI 구성을 Dagger 기반으로
자동화해주는 라이브러리이다.

## Hilt를 왜 쓰는가?

Hilt는 Android 클래스별로 표준 컨테이너를 제공하고, 생명주기에 맞춰 **의존성 그래프**를 관리한다. 즉, 개발자가 직접 객체 생성 코드를 여기저기 흩뿌리지 않아도 되고, Dagger의 컴파일 타임 검증과
런타임 성능 이점도 그대로 가져간다. Android 공식 문서도 Android에서는 Dagger를 직접 구성하기보다 Hilt 사용을 권장한다.

## Hilt Annotation

### @HiltAndroidApp

```kotlin
@HiltAndroidApp
class MyApp : Application()
```

Application 클래스에 붙인다. 이 어노테이션을 기준으로 Hilt의 **애플리케이션 레벨 컴포넌트**가 생성되고, 앱 전체 DI 그래프의 시작점이 된다.

애플리케이션 레벨 컴포넌트

- 앱이 살아있는 동안 유지되는 DI 컨테이너. HiltAndroidApp 어노테이션이 붙으면 Hilt가 내부적으로 DaggerMyApp_HiltComponents_SingletonC같은 클래스 생성

### @AndroidEntryPoint

Activity, Fragment, View, Service 등 Android 프레임워크 클래스에 붙여서 해당 클래스가 Hilt로부터 의존성을 주입받을 수 있게 한다. Hilt는 **Android 클래스별 컨테이너(
Component)**를 제공하므로, 주입 대상 Android 컴포넌트에는 entry point가 필요하다.

```kotlin
@AndroidEntryPoint
class MainActivity : AppCompatActivity()
```

### @Inject

생성자에 붙이면 생성자 주입이고, 필드에 붙이면 필드 주입이다. Android 공식 가이드에서는 가능하면**생성자 주입을 우선**하라고 권장되어 있다. 테스트가 쉽고, 객체 생성 시점에 필요한 의존성이 보장되기
때문.

```kotlin
class UserRepository @Inject constructor(
    private val api: UserApi,
)
```

### @Module

“이 클래스 안에는 DI 설정이 들어있다”는 선언. "이 클래스는 DI 그래프 구성에 사용된다”라는 마커.

자체로는 아무 객체도 만들지 않는다.

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule
```

### @InstallIn

이 모듈이 어느 Hilt 컴포넌트에 설치될지 지정한다. 즉,**어느 생명주기 범위에서 관리할지**결정하는 어노테이션이다. Hilt는 Android 클래스 생명주기에 맞는 컴포넌트를 제공하고, 모듈은 그 중 어디에
속할지 명시해야 한다.

- SingletonComponent : 앱 전체 생명주기와 함께 감
- ActivityRetainedComponent: 구성 변경에도 유지되는 Activity 관련 범위
- ActivityComponent / FragmentComponent: 해당 Android 컴포넌트 생명주기 범위
- ViewModelComponent: ViewModel 생명주기 범위

### @Provides

객체 생성 로직을 직접 작성해야 할 때 사용한다. 주로 외부 라이브러리 클래스처럼 생성자를 수정할 수 없거나 Builder 패턴처럼 생성 과정에 추가 로직이 필요한 경우에 적합하다.

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {

    @Provides
    fun provideRetrofit(): Retrofit {
        return Retrofit.Builder()
            .baseUrl(BuildConfig.BASE_URL)
            .build()
    }
}
```

### @Binds

**인터페이스와 구현체를 연결**할 때 사용한다. 구현체 생성 자체는 @Inject constructor()로 가능하고, “이 인터페이스 요청 시 이 구현체를 써라”만 선언하면 되는 경우 적합하다. 공식 문서도
인터페이스 바인딩에는 @Binds 사용을 권장한다.

```kotlin
interface LoginRepository

class LoginRepositoryImpl @Inject constructor(
    private val api: LoginApi,
) : LoginRepository

@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {

    @Binds
    abstract fun bindLoginRepository(
        impl: LoginRepositoryImpl,
    ): LoginRepository
}
```

### @HiltViewModel

Hilt로 ViewModel 생성자를 주입받게 할 때 사용한다. @HiltViewModel이 붙은 ViewModel은 constructor injection을 통해 의존성을 받을 수 있고,
SavedStateHandle도 함께 주입 가능하다.

```kotlin
@HiltViewModel
class LoginViewModel @Inject constructor(
    private val repository: LoginRepository,
    private val savedStateHandle: SavedStateHandle,
) : ViewModel()
```

---

## 참고 자료

https://developer.android.com/training/dependency-injection/hilt-android?hl=ko&utm_source=chatgpt.com