# 4주차 - Flow

## Flow란?

Flow는 Kotlin Coroutines에서 제공하는 비동기 데이터 스트림 처리 API이다.
시간에 따라 여러 값을 순차적으로 방출(emission)하고 처리할 수 있다.

## 왜 Flow를 사용하는가

- Coroutine 기반으로 suspend / cancellation과 자연스럽게 통합됨
- 비동기 데이터 스트림을 선언적으로 처리 가능
- 다양한 Operator를 통한 데이터 변환/결합 지원
- 생산자/소비자 속도 차이를 자연스럽게 조절
- Android의 Lifecycle, ViewModel, Compose 기반 상태 관리 구조와 쉽게 연계할 수 있음

---

## 구조

Flow는 다음과 같은 3단계 구조로 동작한다.

> Producer → Operator → Consumer
>

```kotlin
flow { emit(1) }      // Producer
    .map { it * 2 }   // Operator
    .collect { }      // Consumer
```

---

## 핵심 특징

### Stream (여러 값의 흐름)

```kotlin
flow {
    emit(1)
    emit(2)
    emit(3)
}
```

- 단일 값이 아니라 시간에 따라 여러 값이 방출됨
- 데이터 흐름을 표현

### Coroutine 기반

```kotlin
flow {
    delay(1000)
    emit(1)
}
```

- suspend 함수 사용 가능
- 코루틴 컨텍스트에서 실행
- non-blocking 처리 가능

### 순차 처리

```kotlin
flow {
    emit(1)
    emit(2)
}.collect {
    delay(100)
}
```

> emit(1) → collect 완료 → emit(2)
>

- 기본적으로 producer / consumer는 순차적으로 실행됨
- 별도 설정 없이는 병렬 처리되지 않음

### 생산/소비 속도 자연 조절

```kotlin
emit(value)
```

- consumer가 느리면 emit이 suspend됨
- 생산자/소비자 속도 자동 동기화

### Cancellation 지원

```kotlin
viewModelScope.launch {
    flow.collect()
}
```

- Coroutine 취소 시 Flow 수집도 함께 취소됨

### Lifecycle 처리

Flow 자체는 Lifecycle-aware 하지 않다.
Android에서는 다음과 같이 lifecycle-safe 하게 수집한다.

```kotlin
lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        flow.collect { }
    }
}
```

- STARTED 상태에서만 collect
- STOPPED 시 자동 cancel

---

## Flow 종류

### 1. Cold Flow (기본)

- 구독(collect)할 때마다 producer가 새롭게 실행됨.
- 각 collector는 독립적인 upstream 실행 인스턴스를 가진다. 즉, upstream이 공유되지 않는다.

사용 사례

- API 호출
- DB Query
- 계산 결과 생성

### 2. Hot Flow

- collector 존재 여부와 무관하게 독립적으로 존재하는 데이터 스트림
- 여러 collector가 동일한 데이터를 공유
- producer, consumer가 분리됨
- upstream 공유 가능

  ### StateFlow

  상태를 표현하기 위한 Hot Flow.

  항상 값이 존재해야 하므로 초기값이 반드시 필요하다.

  과거 모든 값을 저장하지 않고 가장 최근 값 1개만 유지한다.

  상태 최적화를 위해 동일한 값이면 방출이 생략될 수 있다. → UI 상태 관리에 적합하다.

  예) uiState, 로그인 상태, 설정 값 등

  ### SharedFlow

  이벤트를 여러 collector에게 공유해서 전달하는 Hot Flow.

  상태가 아니라 “이벤트 스트림”에 적합

    - 예) Snackbar 표시, navigation 이벤트 등

  replay 매개변수를 통해 최근 이벤트를 몇 개까지 새 collector에게 재전달할지 정할 수 있다.

---

## 주요 Operator

### 변환

map, filter 등등

### 결합

combine: 최신 값을 기반으로 새로운 값 생성

```kotlin
combine(flow1, flow2) { a, b -> a + b }
```

### flatten 계열

flatMapLatest

- 새로운 값이 들어오면 이전 inner Flow를 취소하고 최신 Flow만 collect한다
- 최신 요청만 의미 있는 경우 사용

flatMapConcat

- 각 inner Flow를 순차적으로 모두 실행**.** 이전 Flow가 끝나야 다음 Flow 시작
- 순서 보장, 이전 작업 완료 후 다음 작업 실행할 때 사용

flatMapMerge

- 여러 inner Flow를 동시에 collect.
- 독립적인 병렬 API 호출, 순서 중요하지 않을 때 사용

### collect 계열

collect: 방출된 모든 값을 순차적으로 처리하며, 이전 처리 완료까지 다음 값 대기

collectLatest: 새 값이 방출되면 이전 처리 작업을 취소하고 최신 값만 처리
