# Jetpack Compose Side Effect API

## 1. Side Effect 란?

Composable 함수의 스코프 밖에서 일어나는 상태 변경

e.g. 스낵바 띄우기, 다른 화면 이동, 네트워크 요청

Composable 함수는 이론적으로 Side Effect가 없어야함

- 이유
    - Composable은 언제든 Recomposition 될 수 있음
    - Recomposition 순서는 보장되지 않음
    - Recomposition 중간에 버려질 수 있음 (discarded)
    - 서로 다른 스레드에서 병렬로 실행될 수 있음

이러한 특성으로 인하여 Composable 함수 내에서는 몇 번 실행될지 예측할 수 없는 버그가 일어날 수 있음

## 2. Side Effect API가 필요한 이유

```kotlin
@Composable
fun TempComposable(tempId: String) {
    showSnackBar(tempId)
    Text("Temp: $tempId")
}
```

이 코드는 Recomposition이 일어날 때마다 showSnackBar가 호출되는 형태, 심지어 부모 Composable의 Recomposition에 의해서도 수십 번씩 호출될 수 있음
컴파일러가 Recomposition을 버릴 경우 호출이 안 될 수 있음

따라서 Compose가 제공하는 Side Effect API들은 이런 문제들을 해결함

- 예측 가능한 실행 시점: 컴포지션이 성공적으로 실행된 이후에 실행
- 라이프사이클 인식: Composable이 컴포지션에서 벗어나면 자동으로 정리
- key 기반 재실행: 특정 값이 바뀔 때만 다시 실행되도록 제어

## 3. 주요 Side Effect API 정리

Compose가 제공하는 Side Effect API는 코루틴 기반과 비코루틴 기반으로 나뉨

- LaunchedEffect: 코루틴 기반, 컴포지션 진입 시 suspend 함수 실행
- rememberCoroutineScope: 코루틴 기반, 이벤트 핸들러 등에서 수동으로 코루틴 실행
- rememberUpdatedState: 비코루틴 기반, 재시작 시키지 않고 최신 값을 참조
- DisposableEffect: 비코루틴 기반, 등록 / 해제 가 필요한 리소스 관리
- SideEffect: 비코루틴 기반, Compose 상태를 Compose가 아닌 코드에 반영
- produceState: 코루틴 기반, Compose가 아닌 데이터 소스를 State로 변환 - 내부 구현체에서 LaunchedEffect 사용
- derivedStateOf: 비코루틴 기반, 여러 State를 조합한 파생 State
- snapshotFlow: 비코루틴 기반 (하지만 collect는 필요), State를 Flow로 변환

### LaunchedEffect

Composable의 라이프 사이클 동안 suspend 함수를 실행하고 싶을 때 사용


LaunchedEffect가 컴포지션에 진입하면 람다 블록을 코루틴으로 실행하고 컴포지션을 떠나면 자동으로 취소함, 키 값이 변경되도 재시작됨

- 동작 규칙
    - 컴포지션 진입 시 코루틴 실행
    - 컴포지션 이탈 시 코루틴 취소
    - key가 변경되면 기존 코루틴 취소 후 새 코루틴으로 재시작

### rememberCoroutineScope

LaunchedEffect의 경우 Composable 함수기 때문에 다른 Composable 함수 스코프에서만 사용 가능

이벤트 핸들러, 람다 함수 등에서 코루틴을 실행하고 싶다면 rememberCoroutineScope를 사용

### rememberUpdatedState

LaunchedEffect는 key가 바뀔 때마다 재시작, 최신 값은 참조하고 싶지만 재시작하고 싶지 않을 때 사용

```kotlin
@Composable
fun TimeOutComponent(onTimeOut: () -> Unit) {
    val currentOnTimeout by rememberUpdatedState(onTimeout)

    // currentOnTimeout가 변경되어도 재시작하지 않음
    LaunchedEffect(true) {
        delay(SplashWaitTimeMillis)
        currentOnTimeout()
    }
}
```

### DisposableEffect

등록과 해제가 반드시 필요한 리소스를 다룰 때 사용

e.g. LifecycleObserver, 브로드캐스트 리시버

- 규칙
    - 블록의 마지막은 반드시 onDispose {} 로 끝나야 함 -> 빌드 에러
    - key가 바뀌면 onDispose 호출 후 블록이 다시 실행
    - 컴포지션을 떠날 때도 onDispose 호출

### SideEffect

Compose 상태를 Compose가 관리하지 않는 객체에 공유하고 싶을 때 사용

컴포지션이 성공할 때마다 블록을 실행

```kotlin
@Composable
fun rememberGAComponent(user: User): FirebaseAnalytics {
    val analytics: FirebaseAnalytics = remember { FirebaseAnalytics() }

    SideEffect {
        analytics.setUserProperty("userType", user.userType)
    }
    
    return analytics
}
```

- 주의
  - 매 재구성마다 실행되기 때문에 무거운 작업 x
  - suspend 함수 호출 불가능

### produceState

Flow, LiveData, RxJava, RxKotlin 같은 Compose가 아닌 데이터 소스를 State<T>로 변환할 때 사용

내부적으로 코루틴을 실행하고 결과를 State에 넣어야 함

```kotlin
@Composable
fun <T> produceState(
    initialValue: T,
    producer: suspend ProduceStateScope<T>.() -> Unit,
): State<T> {
    val result = remember { mutableStateOf(initialValue) }
    LaunchedEffect(Unit) { ProduceStateScopeImpl(result, coroutineContext).producer() }
    return result
}
```

- 컴포지션에 진입하면 producer 코루틴 시작
- 컴포지션을 떠나면 코루틴이 취소됨
- 값을 다시 설정해도 Recomposition이 발생하지 않음
- suspend가 아닌 소스를 구독할 때는 awaitDispose {} 로 해제를 설정할 수 있음


### derivedStateOf

여러 State를 조합해서 필요시에만 Recomposition 되는 파생 State를 만들 때 사용

```kotlin
@Composable
fun MessageList(messages: List<Message>) {
    Box {
        val listState = rememberLazyListState()

        LazyColumn(state = listState) { /* ... */ }

        // firstVisibleItemIndex가 계속 변하더라도 > 0 해당 조건만 알면 되기 때문에 재구성 줄이는 용도로 사용
        val showButton by remember {
            derivedStateOf {
                listState.firstVisibleItemIndex > 0
            }
        }

        AnimatedVisibility(visible = showButton) {
            ScrollToTopButton()
        }
    }
}
```

#### 주의

```kotlin
// 나쁜 예
val fullName by remember { derivedStateOf { "$firstName $lastName" } }

// 좋은 예
val fullName = "$firstName $lastName"
```

firstName이나 lastName이 바뀔 때마다 fullName이 바뀌어야 하기 떄문에 derivedStateOf는 쓸모 없는 오버헤드

derivedStateOf는 입력이 출력보다 더 자주 바뀔 때만 의미가 있음


### snapshotFlow

State<T> 를 cold Flow로 변환, 블록 안에서 읽은 State가 바뀌면 새 값 방출

동일한 값은 방출하지 않음

```kotlin
val listState = rememberLazyListState()

LazyColumn(state = listState) { /* ... */ }

LaunchedEffect(listState) {
    snapshotFlow { listState.firstVisibleItemIndex }
        .map { index -> index > 0 }
        .distinctUntilChanged()
        .filter { it }
        .collect {
            // ...
        }
}
```

snapshotFlow는 cold Flow라서 collect를 시작해야 동작함, 보통 LaunchedEffect 내부에서 사용

## 4. Effect Restarting 규칙

LaunchedEffect, DisposableEffect 모두 가변 개수의 key 파라미터를 받음

### Key를 다루는 원칙

- Effect 블록 안에서 사용하는 변수는 key로 넘기는 것을 기본으로 함
- 바뀌어도 재시작하면 안 되는 변수는 rememberUpdatedState로 랩핑
- remember로 감싸서 변하지 않는 값이라면 key로 넘길 필요가 없음
- 호출 지점의 수명을 따라가고 싶다면 true / Unit 같은 상수를 key로 씀

key를 너무 자주 바꾸면 불필요하게 Effect가 재시작되어 성능 저하가 발생하고, 너무 드물게 바꾸면 stale한 값을 참조하는 버그가 생김

## 5. 참고 자료
- [Side-effects](https://developer.android.com/develop/ui/compose/side-effects)

## 6. 문제로 낼만 한거

### 기초
- Jetpack Compose에서 Side Effect란
- Side Effect API를 사용하지 않고 Composable 함수 블록에 사이드 이펙트 발생하는 코드를 작성한 경우 발생하는 문제점
- LaunchedEffect와 rememberCoroutineScope 차이점

### 중급
- LaunchedEffect(Unit) 나 LaunchedEffect(true)의 사용 시점 및 주의점
- TimeOut 값이 변경되어도 람다 재시작을 방지하고 싶은 경우 해결법
- DisposableEffect 사용 시점, LaunchedEffect로 대체가 불가능한지
- SideEffect 와 LaunchedEffect 차이, SideEffect 사용 시점
- derivedStateOf가 오버 헤드로 동작하는 경우 및 derivedStateOf 사용 시기

### 고급
- produceState 내부 구현
- snapshotFlow는 언제 사용하고, cold flow인 이유
- Side Effect API의 재시작을 위해 key를 정의할 때 원칙
- Side Effect는 대부분 ViewModel로 옮길 수 있을 것 같은데 Composable에서 Side Effect API를 써야하는 경우
