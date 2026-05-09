# 6주차 - Coroutine
---

# 1. Coroutine 기본 개념

## Coroutine이란

비동기 작업을 동기 코드처럼 작성할 수 있게 해주는 협력형 실행 단위

```kotlin
viewModelScope.launch {
    val user = api.getUser()   // suspend
    val posts = api.getPosts() // suspend
}
```

- callback 없이 순차 코드 형태 유지
- 내부적으로는 suspend / resume 기반 비동기 처리
- thread를 block하지 않고 작업 중단 및 재개 가능

## Thread와 Coroutine의 차이

| 구분    | Thread               | Coroutine            |
|-------|----------------------|----------------------|
| 실행 단위 | OS 레벨                | Kotlin 런타임           |
| 생성 비용 | 높음                   | 매우 낮음                |
| 전환 비용 | context switch (무거움) | suspend/resume (가벼움) |
| 개수    | 제한적                  | 매우 많음                |
| 스케줄링  | OS                   | Coroutine Dispatcher |

## Coroutine이 경량인 이유

1. 스레드를 직접 생성하지 않음
2. suspend 시 call stack 전체 대신 Continuation으로 필요한 상태만 저장
3. suspend/resume 기반으로 동작하기 때문에 thread switching 비용 감소

## suspend 함수의 의미

중단 가능한 함수

- coroutine 내부에서만 호출 가능
- thread를 block하지 않음
- suspend 시 실행 상태 저장 후 나중에 resume

```kotlin
suspend fun fetch(): User

fun fetch(continuation: Continuation<User>)
```

---

# 2. CoroutineContext

Coroutine 실행 정보를 담는 Key-Value 형태의 컨텍스트

## 주요 구성 요소

### Job

coroutine lifecycle 관리

- cancel / join
- 부모-자식 관계
- 예외 전파

### Dispatcher

어떤 Thread에서 실행할지 결정

### CoroutineName

디버깅용 이름 지정

```kotlin
launch(CoroutineName("Login")) {
    // ...
}
```

### CoroutineExceptionHandler

최상위 coroutine에서 잡히지 않은 exception 처리

```kotlin
val handler = CoroutineExceptionHandler { _, throwable ->
    log(throwable)
}
```

- try-catch로 처리된 예외는 오지 않음
- launch의 최상위 coroutine에서만 의미 있음
- async는 await 시점에 예외 발생

### Context 상속

```kotlin
viewModelScope.launch {
    launch {
        // 부모 context 상속
    }
}
```

- 자식은 부모 context를 그대로 받음
- 자식은 Parent Context + ChildJob으로 Job만 새로 생성하고 나머지는 상속받는다.

---

# 3. Dispatcher

## Dispatchers.Main

- UI Thread(Main Thread)에서 실행
- 단일 스레드, UI 접근 가능
- 오래 걸리는 작업 수행 시 ANR 발생 가능

사용 예시

- UI State 업데이트, 이벤트 처리 등 빠른 작업만 수행해야 함

## Dispatchers.IO

- Blocking I/O 작업용 Thread Pool
- 많은 스레드

사용 예시

- Network
- 파일 읽기/쓰기
- DB 작업

## Dispatchers.Default

- CPU-bound 작업용 Thread Pool
- 병렬 계산 최적화

사용 예시

- JSON 파싱 (대용량)
- 이미지 처리
- 정렬/계산 작업

## withContext

다른 Dispatcher로 context 전환

```kotlin
withContext(Dispatchers.IO) {
    repository.fetch()
}
```

---

# 4. CoroutineScope

## Scope의 역할

Coroutine lifecycle을 관리하는 컨테이너

- coroutine 실행
- 취소 관리
- CoroutineContext 제공
- 구조적 동시성 보장

## viewModelScope

ViewModel 생명주기에 바인딩된 Scope

- ViewModel clear 시 자동 cancel
- 내부적으로 SupervisorJob + Dispatchers.Main 사용

사용 예시

- UI State 관리
- API 호출
- 비즈니스 로직 실행

## lifecycleScope

Activity / Fragment lifecycle에 바인딩된 Scope

- Lifecycle destroy 시 자동 cancel

## rememberCoroutineScope

Compose에서 사용하는 Scope

- Composable lifecycle에 바인딩
- composition에서 제거되면 cancel

사용 예시

- Snackbar
- BottomSheet 제어

## GlobalScope

애플리케이션 프로세스 전체 생명주기를 가지는 전역 CoroutineScope

- 특정 lifecycle에 묶이지 않음
- 부모 Scope 없음
- 앱 프로세스 종료 전까지 살아있을 수 있음

---

# 5. Job과 구조적 동시성

## 구조적 동시성

- Coroutine을 부모 Scope에 귀속시켜 lifecycle과 취소, 에러를 일관되게 관리하는 방식

특징

- 모든 coroutine은 부모를 가짐
- 부모는 자식의 lifecycle을 관리함
- 부모 취소 시 자식도 취소

## 전파 규칙

- 취소: 부모 → 자식 (부모 scope가 cancel 되면 자식 coroutine도 함께 cancel)
- 예외: 자식 → 부모 (자식 예외 발생 시 부모까지 전파되며 부모도 cancel)

## coroutineScope

- 모든 자식이 성공해야 정상 종료되는 Scope
- 자식 하나라도 실패 → 전체 취소

## supervisorScope

- 자식 간 실패를 독립시키는 Scope
- 자식 실패가 부모로 전파되지 않음
- 형제 코루틴 영향 없음

## SupervisorJob

- Job 레벨에서 supervisor 동작 적용

---

# 6. launch vs async

## launch

- Job 반환
- 예외: 즉시 부모로 전파 → 부모 취소
- UI 이벤트 처리, 로깅 등에 사용

## async

- Deferred<T>반환
- await()로 결과 수신
- 예외: 내부에 저장한 뒤, await() 시점에 throw
- 값을 생산하는 작업에 사용

---

# 7. 예외 처리

## try-catch

```kotlin
viewModelScope.launch {
    try {
        repository.fetch()
    } catch (e: Exception) {
        handleError(e)
    }
}
```

- suspend 함수 포함 일반 코드처럼 동작
- 예외를 소비하면 상위 전파되지 않음

## launch 예외 처리

- 예외 발생 시 즉시 부모로 전파 (구조적 동시성 영향 받음)

## async 예외 처리

- 예외를 Deferred에 저장하고, await()시 throw
- await 안 하면 handler로 전달

---

# 8. Cancellation (취소)

- Coroutine은 cooperative cancellation (협력적 취소)방식
- 강제 중단이 아니라 취소 상태를 체크하면서 종료

## 취소 전파

부모 취소 시 자식까지 함께 cancel

```kotlin
val job = viewModelScope.launch {
    launch { delay(1000) }
}

job.cancel() // 자식까지 모두 취소
```

## suspend 함수와 취소

대부분 suspend 함수는 내부적으로 취소 체크 수행

- withContext, delay 등

취소 시 CancellationException 발생

## 취소가 안 되는 케이스

CPU 작업 (suspension point 없음)

```kotlin
while (true) {
    // heavy work
}
```

### 해결 방법

1. isActive 체크
2. yield(): 다른 coroutine에게 실행 양보 + 취소 체크

## finally 블록

```kotlin
try {
    delay(1000)
} finally {
    // 항상 실행됨 (cleanup)
}
```

- cancel 되어도 실행됨
- resource cleanup 용도

## NonCancellable

```kotlin
withContext(NonCancellable) {
    // 반드시 실행해야 하는 작업
}
```

- cancel 상태에서도 실행 보장

---

## 질문 목록

1. suspend 함수는 왜 일반 함수처럼 호출할 수 없나요?
2. Dispatcher를 잘못 쓰면 생기는 문제
3. GlobalScope를 지양하는 이유
4. async 사용 시 await를 호출안하면 뭐가 문제일까요?
5. CancellationException 주의점
6. launch와 async의 차이는 무엇인가요?
7. 구조적 동시성이 무엇인가요?
8. viewModelScope는 왜 SupervisorJob을 사용할까요?
9. withContext와 launch의 차이는 무엇인가요?
