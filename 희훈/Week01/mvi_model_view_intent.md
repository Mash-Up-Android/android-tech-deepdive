# MVI (Model - View - Intent) 아키텍처

## 1. 개요

Model-View-Intent(이하 MVI)는 단방향 데이터 흐름(Unidirectional Data Flow, UDF)와 불변 상태(Immutable State)를 기반으로 하는 반응형 아키텍처 패턴

기존의 Model-View-Controller(MVC), Model-View-Presenter(MVP), Model-View-ViewModel(MVVM) 아키텍처 패턴이 가지고 있던 양방향 데이터 바인딩의 복잡성과 상태 관리 어려움을 해결하기 위해 등장한 아키텍처 패턴

MVI는 Cycle.js 창시자 Andre Staltz가 제안한 개념에서 시작되었으며, 함수형 반응형 프로그래밍 원칙 기반

안드로이드의 경우 Hannes Dorfmann이 차용하여 발전시킴

핵심 철학
- Single Source of Truth: 화면의 상태는 오직 하나의 Model에서만 관리
- 불변성(Immutability): 상태는 직접 변경되지 않고, 새로운 객체가 생성되어 교체됨
- 순수 함수(Pure Function): 상태 변경 로직은 사이드 이펙트가 없는 순수 함수로 구현
- 단방향 순환 - 데이터는 한 방향으로만 흐른다

## 2. MVI 탄생 배경

| 패턴   | 주요 문제점                                              |
|------|-----------------------------------------------------|
| MVC  | Controller가 비대해지는 문제, View와 Model의 양방향 의존성          |
| MVP  | Presenter와 View 사이의 강한 결합                           |
| MVVM | 양방향 데이터 바인딩으로 인한 디버그 난이도 증가, 상태 충돌로 인한 휴먼 에러 가능성 증가 |

### MVI가 해결하고자 하는 문제
1. 상태 충돌 
    - 여러 곳에서 상태를 변경할 경우 예측 불가능한 결과 발생할 가능성 존재
    - 상태를 단일 지점에서 관리하여 문제 차단
2. 스레드 안전성
    - 불변 상태를 사용하기 때문에 멀티스레드 환경에서 안전
3. 디버깅 어려움
    - 단방향 데이터 흐름 때문에 상태 변화 추적이 상대적으로 쉬움
4. 테스트 용이성
    - 순수 함수 기반이기 때문에 단위 테스트가 쉬움

## 3. 핵심 구성 요소

### Model

MVI에서 Model은 화면의 전체 상태(State)를 의미, 화면에 표시되어야 하는 모든 정보가 하나의 불변 객체로 사용됨

```kotlin
data class LoginState(
    val adminUser: Boolean = false,
    val id: String = "",
    val password: String = "",
    val isLoading: Boolean = false,
    val error: String? = null,
    val isLoginSuccess: Boolean = false
)
```

- 특징
   - 불변 객체로 정의
   - View에 사용되는 모든 정보 포함
   - data class의 경우 copy() 함수를 통해 새로운 상태 생성

### View

View는 Model을 화면에 렌더링 해서 사용자가 인식할 수 있게 하는 역할

MVI에서 View는 주어진 상태를 그대로 표현하고 자체적인 상태를 갖지 않는 수동적인 역할

```kotlin
@Composable
fun TempComposableFunction(state: LoginState) {
    // ... state render view logic
}
```

- 특징
    - Model(상태)를 입력으로 받아서 UI를 렌더링
    - 자체적인 상태를 보관하지 않음 (Stateless)
    - 사용자 입력을 Intent로 변환하여 방출

### Intent

Intent는 사용자의 의도를 나타냄

Android의 Intent와는 다른 개념으로, 사용자가 View에서 수행하는 행위를 모델화 시킨 것

```kotlin
sealed interface LoginIntent {
    data class OnEmailValueChanged(
        val input: String,
    ) : LoginIntent

    data class OnPasswordValueChanged(
        val input: String,
    ) : LoginIntent
    
    data object OnClickBack : LoginIntent
    
    data object OnClickConfirm : LoginIntent
}
```

- 특징
    - 사용자의 액션을 불변 객체로 표현
    - View에서 발생한 이벤트를 표현
    - sealed class/interface 를 이용하여 정의

## 4. 데이터 흐름

MVI의 데이터 흐름은 아래와 같은 단방향 순환 구조를 따름

```
User → Intent(Action) -> Business Logic → Model(State) → View → User
```

### 상세 흐름

1. 유저는 뷰를 조작해 행동을 일으킴 (e.g. 클릭, 인풋 입력)
2. 인텐트 생성
3. 인텐트를 비즈니스 로직에서 처리
4. 비즈니스 로직 처리 결과를 바탕으로 새로운 모델 생성
5. 뷰가 생성된 새로운 모델에 대한 화면을 갱신
6. 사용자는 갱신된 뷰를 봄

## 5. 단방향 데이터 흐름 - Unidirectional Data Flow

데이터는 오직 한 방향으로만 흐름: Intent → Model → View 

상태 변경의 원인을 항상 명확하게 추적할 수 있음

### 양방향 데이터 흐름 (e.g. MVVM)

- View가 ViewModel을 변경할 수 있고, ViewModel도 View를 변경할 수 있음
- 데이터 바인딩이 양쪽 모두에서 발생하므로, 상태가 어디서 변경되었는지 추적이 어려움
- 복잡한 화면에서 예측하기 힘든 상태 변화가 발생할 수 있음
    - 다중 상태 값이 발생할 수 있음

## 6. Reducer

Reducer는 MVI의 핵심 로직이며, 현재 상태와 인텐트를 입력으로 받아서 새로운 상태를 리턴하는 순수 함수

```kotlin
fun reduce(
    state: LoginState,
    intent: LoginIntent,
): LoginState = when (intent) {
    is LoginIntent.OnEmailValueChanged -> {
        state.copy(id = intent.input)
    }

    is LoginIntent.OnPasswordValueChanged -> {
        state.copy(password = intent.input)
    }

    is LoginIntent.OnClickConfirm -> {
        state.copy(isLoading = true)
    }

    is LoginIntent.OnClickBack -> {
        // impl side effect
    }
}  
```

- Reducer 규칙
    - 동일한 입력에 대해 항상 동일한 값을 리턴하는 순수 함수여야함
    - API 호출, DB 접근 등 사이드 이펙트를 포함하지 않음
    - 현재 상태를 새로 생성해서 immutability를 지켜야함

## 7. Android 에서 MVI

### 기본 구조
```kotlin
// 모델
data class State(
    val isLoading: Boolean = false,
    val tempData: String? = null,
)

sealed interface Intent {
    data object OnClickTemp : Intent
}

sealed interface SideEffect {
    data object NavigateToTemp : SideEffect
}

// 비니지스 로직 및 인텐트 처리
class TempViewModel : ViewModel() {
    private val _state: MutableStateFlow<State> = MutableStateFlow(State())
    val state: StateFlow<State> = _state.asStateFlow()
    
    // Channel or SharedFlow
    private val _sideEffect: Channel<SideEffect> = Channel()
    val sideEffect: Flow<SideEffect> = _sideEffect
    
    // ...
    
    fun reduce(
        state: State,
        intent: Intent,
    ) {
        // ...
    }
    
    fun postSideEffect() {
        // ...
    }
}

// 뷰
@Composable
fun TempRoute() {
    val viewModel: TempViewModel = viewModel()
    val state by viewModel.state.collectAsState()
    
    // ...
}
```

## 8. 상태 관리

MVI에서는 두 가지 종류의 상태를 구분

### UI State (지속 상태 - reducer)

- 화면에 지속적으로 갱신되는 상태
- StateFlow, LiveData 등으로 관리 

### SideEffect (일회성 이벤트) 

- 한 번만 실행되어야 하는 이벤트
- Channel, SharedFlow 등으로 관리

### 불변 상태의 이점

- 이전 상태와 새 상태의 비교가 쉬움
- 현 상태와 이전 인텐트를 알고 있으면 과거로 되돌리기가 쉬움
- 동시 접근 시에도 데이터 정합성이 보장됨
- 상태 변화를 예측할 수 있음

## 9. SideEffect 처리

순수 함수인 Reducer 내부에서 처리할 수 없는 작업

e.g. 네비게이션

### 처리 전략

- Channel
   - 구독자가 존재할 때 값을 방출

- SharedFlow
   - 구독자가 존재하지 않아도 값을 방출

## 10. MVI 장점

- 상태를 예측 가능함
- reducer가 순수 함수기 때문에 테스트가 용이
- 멀티스레드 안전성
- 다중 상태로부터 발생하는 오류가 현저히 적음

## 11. MVI 단점

- 간단한 화면에도 많은 코드가 필요 (보일러플레이트)
- 함수형 프로그래밍, 반응형 스트림, 상태 머신 등 개념을 학습해야할 것이 많아서 러닝 커브가 있음
- 객체 생성시 발생하는 오버헤드 (불변 객체 사용으로 인한 객체 생성)

## 12. 참고 자료

- [Andre Staltz - Unidirectional Data Flow](https://staltz.com/unidirectional-user-interface-architectures.html)
- [Hannes Dorfmann - 반응형 앱 MVI](https://hannesdorfmann.com/android/mosby3-mvi-1/)
- [Android 공식 문서 - 앱 아키텍처](https://developer.android.com/topic/architecture)
- [Android 공식 문서 - Jetpack Compose UDF 흐름도](https://developer.android.com/develop/ui/compose/architecture)
- [Github Orbit-MVI](https://github.com/orbit-mvi/orbit-mvi)
