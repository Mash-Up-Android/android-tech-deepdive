              # 5주차 - SideEffect

SideEffect는 Composable의 순수성을 유지하면서, 외부 상태와의 상호작용을 lifecycle 기반으로 안전하게 실행하기 위한 API이다.

- Composable의 순수성: 같은 입력(State)을 받으면 항상 같은 UI를 생성하고, 외부 상태를 변경하지 않는 성질 (pure function)

---

## 1. 왜 SideEffect API가 필요한가

Compose는 기본적으로 다음 원칙을 가진다.

- Composable은 순수 함수처럼 동작
- 같은 입력 → 같은 UI
- recomposition은 언제든지 발생 가능

```kotlin
@Composable
fun Screen(viewModel: MyViewModel) {
    viewModel.loadData() // ❌ 문제
}
```

이 코드는 recomposition마다 API를 계속 호출한다. 즉, UI 재구성과 비즈니스 로직 재실행은 분리되어야 한다.

→ 그래서 **SideEffect API로** side effect의 실행 기준을 "recomposition"이 아니라 "key 변화, composition 진입/이탈" 같은 명확한 lifecycle로 옮긴다.

---

## 2. SideEffect API 전체 구조

Compose에서 제공하는 SideEffect는 목적별로 나뉜다.

### 실행 시점 기준

| API                      | 실행 시점                        |
|--------------------------|------------------------------|
| `LaunchedEffect`         | composition 진입 시 (Coroutine) |
| `rememberCoroutineScope` | 이벤트 기반 실행                    |
| `SideEffect`             | recomposition commit 이후      |
| `DisposableEffect`       | lifecycle (onDispose 포함)     |
| `produceState`           | 외부 → Compose state 변환        |
| `derivedStateOf`         | state 파생 계산                  |
| `snapshotFlow`           | Compose state → Flow 변환      |

---

## 3. 핵심 API 정리

### 3.1 LaunchedEffect

**Composition에 맞춰 Coroutine 실행**

```kotlin
LaunchedEffect(key) {
    // suspend 가능
}
```

- 최초 composition 진입 시 실행
- key 변경 시 cancel 후 재실행
- 내부적으로`rememberCoroutineScope + launch` 구조

**사용 케이스**

- API 호출
- Flow collect
- delay 기반 로직

```kotlin
LaunchedEffect(Unit) {
    viewModel.loadData()
}
```

→ 최초 1회 실행

---

### 3.2 rememberCoroutineScope

**사용자 이벤트 기반 Coroutine 실행**

```kotlin
val scope = rememberCoroutineScope()

Button(onClick = {
    scope.launch {
        snackbarHostState.showSnackbar("완료")
    }
})
```

- recomposition 영향 없음
- lifecycle은 composition에 종속됨

**핵심 차이**

- `LaunchedEffect`: 자동 실행
- `rememberCoroutineScope`: 수동 실행

---

### 3.3 SideEffect

**Recomposition commit 이후 실행 (메인스레드)**

```kotlin
SideEffect {
    analytics.setUserProperty("screen", "home")
}
```

- Composition 변경 사항이 실제 UI 트리에 적용된 이후 실행됨
- suspend 함수 사용 불가 (동기 실행)
- 매 recomposition마다 실행되므로 비용이 큰 작업에는 부적절

**사용 케이스**

- 외부 SDK sync (Analytics, Logging 등)

---

### 3.4 DisposableEffect

**Lifecycle-aware side effect**

    DisposableEffect(key1) {
        val observer = ...
    
        onDispose {
            observer.remove()
        }
    }

- key 변경 또는 composition 제거 시 cleanup 실행

**사용 케이스**

- Listener 등록/해제
- BroadcastReceiver
- LifecycleObserver

#### 왜 필요하냐?

- Composable은 제거되는 시점을 직접 감지할 수 없기 때문에 cleanup을 수행할 수 없다. 따라서**cleanup 지점**이 필요

---

### 3.5 produceState

**외부 비동기 데이터를 Compose State로 변환**

```kotlin
val state by produceState(initialValue = 0) {
    value = repository.load()
}
```

- 내부적으로 `LaunchedEffect + mutableStateOf`
- `value` 변경 → recomposition 발생

**사용 케이스**

- ViewModel 없이 간단한 비동기 데이터를 UI에서 직접 다룰 때 사용
- 복잡한 로직은 ViewModel에서 처리하는 것이 권장됨

---

### 3.6 derivedStateOf

**불필요한 recomposition 방지용 파생 상태**

```kotlin
val isEnabled by remember {
    derivedStateOf { input.length > 3 }
}
```

- 입력 state가 변해도 계산 결과가 동일하면 recomposition이 발생하지 않음
- 즉, 불필요한 recomposition을 줄이기 위한 memoization 역할

---

### 3.7 snapshotFlow

**Compose State → Flow 변환**

```kotlin
LaunchedEffect(Unit) {
    snapshotFlow { text }
        .collect { /* 처리 */ }
}
```

- Compose state를 읽는 block을 Flow로 변환
- 해당 state가 변경될 때마다 새로운 값 emit
- 내부적으로 snapshot 시스템 기반

---

| 항목        | LaunchedEffect                       | rememberCoroutineScope                      |
|-----------|--------------------------------------|---------------------------------------------|
| 실행 시점     | Composition 진입 시 자동 실행               | 필요한 시점에 직접 launch                           |
| lifecycle | Composition에 종속                      | Composition에 종속                             |
| cancel    | key 변경 또는 Composition 이탈 시 자동 cancel | Composition 이탈 시 자동 cancel, 그 전까지는 직접 제어 가능 |
| 주 용도      | 초기 로드, collect, key 기반 재실행           | 클릭, 스와이프, 사용자 액션 기반 coroutine 실행            |

---

### 4.2 SideEffect vs LaunchedEffect

| 항목      | SideEffect                        | LaunchedEffect                       |
|---------|-----------------------------------|--------------------------------------|
| suspend | 불가                                | 가능                                   |
| 실행 타이밍  | recomposition commit 이후 (UI 반영 후) | Composition에 들어온 직후 coroutine launch |
| 실행 스레드  | Main (즉시 실행)                      | Coroutine (비동기)                      |
| 재실행 조건  | recomposition 발생 시 매번             | key 변경 시 재실행                         |
| 용도      | 외부 객체와 값 동기화                      | 비동기 작업 / Flow / one-shot effect      |

---

## 5. 아키텍처 관점 정리

Compose + MVI 기준

    ViewModel (State + Business Logic)
        ↓
    Composable (Render)
        ↓
    SideEffect (UI 내부에서 lifecycle 기반 실행)

### 역할 분리

| 레이어            | 역할                                |
|----------------|-----------------------------------|
| ViewModel      | business logic, state 관리          |
| Composable     | state 기반 UI rendering             |
| SideEffect API | UI lifecycle 기반 side effect 실행 제어 |

---

## 6. 정리

> Compose SideEffect API는 recomposition 과정에서 발생할 수 있는 부수효과를 제어하기 위해, side effect의 실행을 composition lifecycle 기준으로 분리하여
> 안전하게 수행하도록 하는 메커니즘이다.
