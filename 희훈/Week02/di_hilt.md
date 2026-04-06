# Dependency Injection - Hilt

## 1. 의존성 주입 - Dependency Injection

### 기본 개념
의존성 주입(Dependency Injection, DI)은 객체가 필요로하는 의존 객체를 직접 생성하지 않고 외부에서 전달받는 설계 패턴

### DI를 사용하지 않는 경우
```kotlin
class TempRepository {
    private val testLocalDataSource = TestLocalDataSource(
        dao = TestDatabase.getInstance().testDao()
    )
}
```

코드 문제점

1. TempRepository 가 TestLocalDataSource 와 TestDatabase 를 모두 알고 있어야 함
2. TempRepository 를 테스트하기 위해 testLocalDataSource 를 임의 값을 넣을 수 없음
3. TestLocalDataSource 구현이 변경되면 모든 곳을 다 변경해야함

### DI를 적용한 경우
```kotlin
class TempRepository(
    private val testLocalDataSource: TestLocalDataSource,
) {
    // ...
}
```

- TempRepository 는 TestLocalDataSource 이 어떻게 만들어지는지 알 필요가 없음
- 테스트 시에 임의의 TestLocalDataSource를 주입할 수 있음

## 2. Hilt

### Dagger
상당한 양의 보일러플레이트 코드가 필요함
Component 정의, SubComponent 구성, 수동 빌드 등 반복해서 해야하는 작업이 많음

### Dagger Hilt
Dagger 기반으로 구축된 라이브러리로 DI를 제공

- 보일러플레이트 제거: 어노테이션만으로 자동 생성
- 표준화된 Component: Android 생명주기에 맞춰진 정의된 Component 제공
- 컴파일 타임 검증: 런타임 에러 방지

### Dagger vs Hilt vs Koin

| 항목    | Dagger | Hilt | Koin (Koin-Annotation) |
|-------|--------|------|------------------------|
| 검증 시점 | 컴파일    | 컴파일  | 런타임 (컴파일)              |
| 코드 생성 | O      | O    | X (O)                  |
| 성능    | 좋음     | 좋음   | 상대적으로 안좋음 (보통)         |
| 러닝 커브 | 높음     | 중간   | 낮음 (중간)                |

## 3. Hilt KSP
기존 KAPT 사용 -> KSP 로 마이그레이션
- 2배 정도 빠른 빌드 속도

## 4. Hilt 어노테이션

### 진입점 관련
- `@HiltAndroidApp`: `Application` 클래스에 선언, Hilt 코드 생성 시작점, 앱 DI 컨테이너 생성
- `@AndroidEntryPoint`: Activity, Fragment, View, Service, BroadcastReceiver 에 선언, 해당 Android 클래스에서 주입 활성화
- `@HiltViewModel`: ViewModel 에 선언, ViewModel에 대한 주입 활성화

### 의존성 관련
- `@Inject`: 생성자/필드/매서드에 붙여 Hilt 주입 표시
- `@Module`: Hilt에 바인딩 정보를 제공하는 클래스에 사용
- `@InstallIn`: Module의 Component Scope를 지정
- `@Provides`: Module 내에서 인스턴스 생성 방법 정의 (팩토리)
- `@Binds`: 인터페이스 구현체 맵핑 정의

## 5.Hilt Component 계층 구조
Android 생명주기에 맞춰진 정의된 Component 제공

### Component 종류
- SingletonComponent: Application onCreate시 생성, Application 소멸시 소멸
- ActivityRetainedComponent: Activity onCreate시 생성, Activity onDestroy시 소멸 (구성 변경 무관)
    - 부모: SingletonComponent
- ServiceComponent: Service onCreate시 생성, Service onDestroy시 소멸
    - 부모: SingletonComponent
- ViewModelComponent: ViewModel 생성시 생성, ViewModel onCleared시 소멸
    - 부모: ActivityRetainedComponent
- ActivityComponent: Activity onCreate시 생성, Activity onDestroy시 소멸
    - 부모: ActivityRetainedComponent
- FragmentComponent: Fragment onAttach시 생성, Fragment onDestroy시 소멸
    - 부모: ActivityComponent
- ViewComponent: View 생성시 생성, View 소멸시 소멸 
    - 부모: ActivityComponent
- ViewWithFragmentComponent: View `@WithFragmentBindings`, View 생성시 생성, View 소멸시 소멸
    - 부모: FragmentComponent

### 기본 바인딩
각 Component는 기본적으로 바인딩을 제공

- SingletonComponent -> Application
- ActivityComponent -> Activity
- FragmentComponent -> Fragment

## 6. Scope의 생명주기
Scope는 의존성의 인스턴스 수명 결정, Scope를 지정하지 않으면 주입시마다 새 인스턴스 생성됨(UnScoped)

- `@Singleton`: SingletonComponent, 앱 전체
- `@ActivityRetainedScoped`: ActivityRetainedComponent, 구성 변경에도 유지
- `@ServiceScoped`: ServiceComponent, Service 수명
- `@ViewModelScoped`: ViewModelComponent, ViewModel 수명
- `@ActivityScoped`: ActivityComponent, Activity 수명
- `@FragmentScoped`: FragmentComponent, Fragment 수명
- `@ViewScoped`: ViewComponent, View 수명

### Scoped vs Unscoped
```kotlin
// Unscoped — 주입할 때마다 새 인스턴스 생성
class AnalyticsLogger
@Inject constructor() { /* ... */ }

// Scoped — SingletonComponent 내에서 하나의 인스턴스만 유지
@Singleton
class AppDatabase
@Inject constructor(
    @ApplicationContext context: Context
) { /* ... */ }
```

- 불필요한 Scope는 메모리 누수의 원인이 될 수 있음

## 7. Module과 Binding - `@Provides` vs `@Binds`

### Module 필요 시점
`@Inject` 생성자만으로 바인딩을 정의할 수 없는 경우

- 인터페이스: 구현체가 어떤 것임을 알려줘야함
- 외부 라이브러리 클래스: 해당 프로젝트에서 구현체 코드를 소유하지 않는 클래스
- 빌더 패턴으로 생성해야 하는 객체: 단순 생성자 호출이 아닌 경우

### `@Provides`
```kotlin
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    @Provides
    @Singleton
    fun provideOkHttpClient(): OkHttpClient {
        return OkHttpClient.Builder()
            .connectTimeout(30, TimeUnit.SECONDS)
            .addInterceptor(HttpLoggingInterceptor())
            .build()
    }
}
```

- 인스턴스 생성 로직 직접 작성

### `@Binds`
```kotlin
interface UserRepository {
    suspend fun getUser(id: String): User
}

class UserRepositoryImpl
@Inject constructor(
    private val apiService: ApiService,
    private val userDao: UserDao
) : UserRepository {
    override suspend fun getUser(id: String): User { 
        // ...
    }
}

@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {

    @Binds
    @Singleton
    abstract fun bindUserRepository(
        impl: UserRepositoryImpl
    ): UserRepository
}
```

- 인터페이스와 구현체 사이 매핑

### `@Provides` vs `@Binds` 선택 기준

- 인터페이스와 구현체 사이 매핑 -> `@Binds`
- 외부 라이브러리 클래스 인스턴스 생성 -> `@Provides`
- 빌더 패턴으로 생성하는 객체 -> `@Provides`
- 커스텀 로직 생성 필요 -> `@Provides`

### 제약 사항
- `@Binds`는 abstract 함수로 정의해야 하므로 Module 클래스도 `abstract class`여야 함
- @Provides`는 실제 바디가 있는 함수여서 `object` 클래스에 작성

## 8. Qualifier
같은 타입의 의존성을 여러 개 제공해야 할 때 `@Qualifier`를 사용

### 커스텀 Qualifier 정의
```kotlin
@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class AuthInterceptorOkHttpClient

@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class OtherInterceptorOkHttpClient
```

### Module에서 Qualifier 사용
```kotlin
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {

    @AuthInterceptorOkHttpClient
    @Provides
    @Singleton
    fun provideAuthOkHttpClient(
        authInterceptor: AuthInterceptor
    ): OkHttpClient {
        return OkHttpClient.Builder()
            .addInterceptor(authInterceptor)
            .build()
    }

    @OtherInterceptorOkHttpClient
    @Provides
    @Singleton
    fun provideOtherOkHttpClient(): OkHttpClient {
        return OkHttpClient.Builder()
            .build()
    }
}
```

### 주입시 Qualifier 지정
```kotlin
class TestRepository
@Inject constructor(
    @AuthInterceptorOkHttpClient private val okHttpClient: OkHttpClient
)
```

## 9. ViewModel 주입 — `@HiltViewModel`

### 선언
```kotlin
@HiltViewModel
class TempViewModel
@Inject constructor(
    // ...
) : ViewModel() {
    // ...
}
```

### Activity/Fragment에서 ViewModel 사용
```kotlin
@AndroidEntryPoint
class TempActivity : AppCompatActivity() {
    private val viewModel: TempViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // ...
    }
}
```

### Compose에서 ViewModel 사용
```kotlin
@Composable
fun TempScreen(
    viewModel: TempViewModel = hiltViewModel(),
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    // ...
}
```

## 10. 참고자료
- [hilt-android](https://developer.android.com/training/dependency-injection/hilt-android)
- [hilt-jetpack](https://developer.android.com/training/dependency-injection/hilt-jetpack)
- [dagger-hilt](https://dagger.dev/hilt/)

## 11. 문제로 낼만 한거

### 기본
- DI가 무엇인지
- DI를 적용하지 않았을 때의 문제점
- Dagger, Hilt, Koin의 차이점은 무엇인지 
- Dagger, Hilt, Koin을 각각 어떤 상황에서 선택할건지
- `@Inject`, `@Module`, `@Provides`, `@Binds`의 역할을 각각 설명

### Component, Scope
- Hilt의 Component 계층 구조를 설명
- `SingletonComponent`와 `ActivityRetainedComponent`의 차이
- Scoped와 Unscoped 바인딩의 차이
- 불필요한 Scope 지정이 문제가 되는 이유
- `ActivityComponent`, `ActivityRetainedComponent` 두 컴포넌트는 Configuration Change 시 어떻게 동작하는지

### Module
- `@Provides` 와 `@Binds` 차이점
- `@Binds`를 쓸 수 있는데 `@Provides`를 쓰면 어떤 차이가 있는지
- 같은 타입의 의존성을 여러 개 제공해야 할 때 어떻게 하는지
- `@Qualifier`를 사용한 실제 예시
- `@Binds` 모듈은 왜 `abstract class`여야 하고 `@Provides` 모듈은 왜 object로 작성해야 하는지
