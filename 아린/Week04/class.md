## 1. data class

값(데이터) 보관이 목적인 클래스

컴파일러가 `equals`, `hashCode`, `toString`, `copy`, `componentN` 함수를 자동으로 생성해줌


상속 불가하다.

```kotlin
data class User(
    val id: Int,
    val name: String,
    val email: String
)

val user = User(1, "기마딘", "arin@example.com")

// 이메일만 바꾼 새 인스턴스 생성
val updated = user.copy(email = "new@example.com")

// 구조 분해
val (id, name, _) = user
```

Compose UiState 패턴

```kotlin
// 모든 필드에 기본값 → 초기 상태를 따로 만들 필요 없음
data class HomeUiState(
    val items: List<Item> = emptyList(),
    val isLoading: Boolean = false,
    val errorMsg: String? = null
)

class HomeViewModel : ViewModel() {
    private val _uiState = MutableStateFlow(HomeUiState())
    val uiState = _uiState.asStateFlow()

    fun setLoading() {
        // copy()로 새 인스턴스를 만들어야 StateFlow가 변경을 감지함
        // 기존 객체 필드를 직접 바꾸면 참조가 그대로라 감지 안 됨
        _uiState.update { it.copy(isLoading = true) }
    }
}
```

Room Entity 활용

```kotlin
// data class + @Entity 조합이 기본 패턴
// equals/hashCode가 자동 생성되므로 DB 조회 결과 비교도 편함
@Entity(tableName = "products")
data class Product(
    @PrimaryKey val productId: Int,
    val title: String,
    val price: Double
)
```

Q. `equals` 비교 기준은?

A. primary constructor에 선언된 프로퍼티만 비교함
바디에 선언된 프로퍼티는 포함되지 않음

---

Q. Compose에서 `copy()`를 쓰는 이유는?

A. Compose는 State의 참조가 바뀌어야 리컴포지션이 발생함
`copy()`는 새 인스턴스를 반환하기 때문에 불변성을 유지하면서 상태 변경을 트리거할 수 있음
기존 객체의 필드를 직접 수정하면 참조가 그대로라 Compose가 변경을 감지하지 못함

---

Q. `data class`를 상속할 수 없는 이유는?

A. 자동 생성된 `equals`/`hashCode`가 상속 계층에서 리스코프 치환 원칙을 위반할 수 있기 때문
부모가 `data class`면 자식의 `equals` 구현이 일관성을 보장하기 어려워서 언어 차원에서 막아놨음



## 2. sealed class / sealed interface

하위 타입을 같은 파일로 제한하는 클래스

`when` 식에서 모든 서브타입을 분기로 처리하면 `else` 없이 컴파일이 통과됨

새 서브클래스를 추가하면 처리 안 한 `when` 분기를 컴파일 에러로 잡아줘서 누락 방지에 좋음

`sealed class`는 단일 상속만 가능하고
`sealed interface`는 다중 구현이 가능함

```kotlin
// out T — 공변성 선언
sealed interface UiState<out T> {
    data object Loading : UiState<Nothing>
    data class  Success<T>(val data: T) : UiState<T>
    data class  Error(val msg: String) : UiState<Nothing>
}

@Composable
fun Screen(state: UiState<List<Item>>) {
    // else 없이 처리
    // Error 케이스를 빠뜨리면 컴파일 에러 발생
    when (state) {
        is UiState.Loading  -> CircularProgressIndicator()
        is UiState.Success  -> ItemList(state.data)
        is UiState.Error    -> ErrorText(state.msg)
    }
}
```

Navigation 경로 정의

```kotlin
sealed class Screen(val route: String) {
    data object Home : Screen("home")
    data object Detail : Screen("detail/{id}") {
        // route 문자열에서 실제 id를 채워주는 헬퍼
        fun createRoute(id: Int) = "detail/$id"
    }
}

navController.navigate(Screen.Detail.createRoute(42))
```


## 3. enum class

정해진 상수 집합을 표현하는 클래스

각 상수는 싱글턴이고 `ordinal`, `name` 프로퍼티와 `entries`, `valueOf()`를 기본으로 제공함

프로퍼티와 메서드를 가질 수 있고
`abstract` 메서드를 선언하면 각 상수가 이를 구현해야 함

```kotlin
// 세미콜론 — 상수 목록의 끝을 알림. 그 아래에 멤버가 있을 때 필요
enum class HttpStatus(val code: Int, val message: String) {
    OK(200, "Success"),
    NOT_FOUND(404, "Not Found"),
    SERVER_ERROR(500, "Internal Error");

    fun isSuccess() = code in 200..299
}
```

Compose 테마 전환

```kotlin
enum class AppTheme { LIGHT, DARK, SYSTEM }

@Composable
fun ThemeSelector(current: AppTheme, onChange: (AppTheme) -> Unit) {
    // entries — values() 대신 사용. 불변 List 반환
    AppTheme.entries.forEach { theme ->
        RadioButton(
            selected = theme == current,
            onClick = { onChange(theme) }
        )
    }
}
```

**면접 예상 질문**

Q. `values()`와 `entries`의 차이는?

A. `values()`는 호출할 때마다 새 배열을 생성함

Kotlin 1.9+의 `entries`는 불변 `List`를 반환하며 재사용되기 때문에 성능이 더 좋음

신규 코드라면 `entries` 쓰는 게 권장됨

---

Q. `enum class`에서 `abstract` 메서드를 정의할 수 있는가?

A. 가능함
각 상수가 반드시 구현해야 함
근데 상수별 동작이 복잡해진다면 `sealed class`로 대체하는 게 나음

---

Q. `enum class`가 `sealed class`보다 적합한 경우는?

A. 상수마다 다른 데이터 구조가 필요 없고
방향·HTTP 메서드·테마처럼 고정된 집합을 표현할 때
그리고 `ordinal`, `name`, `entries` 같은 enum 기본 기능이 필요할 때

--- 

Q. `sealed class`와 `enum class`의 결정적 차이는?

A. `enum`은 각 상수가 동일한 타입의 인스턴스 하나뿐임
`sealed class`는 서브클래스마다 서로 다른 프로퍼티와 인스턴스를 가질 수 있어서 상태별로 다른 데이터를 담을 수 있음
예를 들어 `Error`에는 메시지를, `Success`에는 실제 데이터를 담듯 각각 다르게 실을 수 있음

## 4. object / companion object / data object

`object` 선언은 클래스 정의와 동시에 싱글턴 인스턴스를 생성함

`companion object`는 클래스 내부의 정적 멤버 역할을 하지만 실제로는 객체 인스턴스
`data object`는 
일반 `object`에 `toString()`, `equals()`, 직렬화 지원이 추가된 형태
`sealed` 내부의 무상태 케이스에 주로 사용함

```kotlin
// 앱 전역 설정값 보관용 싱글턴
object NetworkConfig {
    const val BASE_URL = "https://api.example.com"
    val timeout = 30_000L
}
```

```kotlin
// private constructor + companion object 팩토리 패턴
// 외부에서 new 대신 create()만 쓰도록 강제
class Repository private constructor(val db: Database) {
    companion object {
        fun create(context: Context) = Repository(Room.build(context))
    }
}

val repo = Repository.create(context)
```

```kotlin
// data object — sealed 안에서 무상태 케이스에 씀
// 일반 object였다면 toString()이 "Loading@1a2b3c" 같은 주소값으로 나옴
sealed interface UiState {
    data object Loading : UiState
    data class  Success(val data: List<Item>) : UiState
    data class  Error(val msg: String) : UiState
}
```

**면접 예상 질문**

Q. `object`는 스레드 안전한가?

A. JVM 클래스 로딩이 초기화를 보장하므로 기본적으로 스레드 안전함
내부에 mutable 상태가 있다면 별도 동기화가 필요함

---

Q. `data object`를 쓰는 이유는?

A. 일반 `object`는 `toString()`이 의미 없는 주소값을 반환함
`data object`는 클래스명("Loading")을 반환하고 직렬화도 지원함
로그 찍을 때나 직렬화할 때 훨씬 편함

## 5. value class

단일 프로퍼티를 래핑하는 경량 클래스

컴파일 후 대부분의 경우 래퍼 객체 생성 없이 내부 타입으로 대체(인라인)됨

`@JvmInline` 어노테이션이 필수

Compose의 `Dp`, `Color`, `Sp`가 내부적으로 `value class`로 구현되어 있음

boxing이 발생하는 경우 세 가지

- nullable 타입(`UserId?`)으로 사용할 때
- 제네릭 타입 파라미터로 사용할 때
- 인터페이스 타입으로 사용할 때


**면접 예상 질문**

Q. `value class`가 boxing되는 경우는?

A. nullable 타입(`UserId?`)으로 사용할 때

제네릭 타입 파라미터로 전달될 때

인터페이스 타입으로 사용될 때

---

Q. Compose의 `Dp`, `Color`가 `value class`인 이유는?

A. 매우 빈번하게 생성되는 값이라 `value class`로 선언하면

대부분 primitive로 인라인되어 GC 부하 없이 타입 안전성을 확보할 수 있음

`Dp`는 `Float`로, `Color`는 `Long`으로 인라인됨

## 6. abstract class vs interface

`abstract class`는 상태(필드)와 생성자를 가질 수 있으며 단일 상속

`interface`는 다중 구현이 가능하고 default 메서드를 제공할 수 있음

실무에서는 Repository나 UseCase의 계약을 `interface`로 정의하고
테스트 시 Fake 구현체로 교체하는 패턴이 일반적

`abstract class`는 `BaseViewModel`처럼 여러 ViewModel이 공통 상태나 로직을 공유할 때 사용함

```kotlin
// interface로 계약만 정의
interface UserRepository {
    suspend fun getUser(id: Int): User
    suspend fun saveUser(user: User)
}

// 테스트용 Fake — 네트워크/DB 없이 ViewModel 테스트 가능
class FakeUserRepository : UserRepository {
    override suspend fun getUser(id: Int) = User(id, "테스트유저")
    override suspend fun saveUser(user: User) { /* 없음 */ }
}
```


**면접 예상 질문**

Q. Repository 계층에서 `interface`를 쓰는 이유는?

A. 테스트 시 `FakeRepository`로 교체할 수 있어 ViewModel 단위 테스트가 쉬워짐

Hilt 등 DI 프레임워크와도 자연스럽게 연동됨

구현을 몰라도 계약(함수 시그니처)만 보고 사용 가능해서 모듈 간 결합도가 낮아짐

---

Q. Kotlin `interface`에서 상태를 가질 수 있는가?

A. 프로퍼티를 선언할 수는 있지만 backing field를 가질 수 없음

구현 클래스가 backing field를 제공해야 함

`abstract class`는 직접 상태를 보유할 수 있음

---

Q. `abstract class`와 `interface` 중 어떤 걸 써야 하는가?

A. 상태 공유나 공통 로직이 필요하면 `abstract class`

계약 정의나 다중 구현이 목적이면 `interface`

가능하면 `interface`를 우선 고려하는 게 일반적인 설계 방향

