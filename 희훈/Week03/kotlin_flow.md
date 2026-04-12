# Kotlin Flow

## 1. Flow
Kotlin Flow는 코루틴을 기반으로 만들어진 비동기 데이터 스트림

suspend 함수가 단일 값만 리턴하는 것과 다르게 여러 값을 순차적으로 emit할 수 있음

개념적으로는 Iterator 나 Sequence 와 비슷하지만, suspend 함수를 활용하기 때문에 메인 스레드를 블록하지 않고 비동기 작업을 안전하게 수행할 수 있음

### 대표적인 특징
- 비동기 스트림: 코루틴 기반으로 동작하여 여러 값을 순차적으로 emit
- Cold 스트림: 기본적으로 collect가 일어나기 전까지 실행되지 않음
- 타입 안정성: 제네릭 타입이 명시됨
- 취소 가능: 코루틴 기반이라 취소가 가능

## 2. Flow 구성 요소

### 생산자 - producer
스트림에 추가될 데이터를 생성

e.g. flow 빌더 블록 내부

### 중개자 - intermediate
방출되는 각 값을 변경하거나 반환

e.g. map, filter

### 소비자 - consumer
스트림에서 값을 사용

e.g. collect

## 3. Cold Flow vs Hot Flow

### Cold Flow
Flow는 기본적으로 cold 스트림, collect가 호출되기 전까지 producer 코드가 실행되지 않음

여러 collector가 collect하면 각각 독립적으로 producer 실행

### Hot Flow
StateFlow, SharedFlow는 Hot 스트림

collector 유무에 상관없이 인스턴스가 존재하며 여러 collector에 동일한 값을 broadcast 함

## 4. Flow 생성 방법

### flow {}
가장 기본적인 형태, 블록 내부에서 emit을 이용해 방출

```kotlin
val flow: Flow<Int> = flow {
    for (i in 1..5) {
        delay(100)
        emit(i)
    }
}
```

### flowOf()
고정된 값들로 Flow 방출

```kotlin
val flow = flowOf("A", "B", "C")
```

### asFlow()
컬렉션, 시퀀스, 배열 등을 flow로 변환

```kotlin
val flow1 = (1..3).asFlow()
val flow2 = listOf("a", "b", "c").asFlow()
```

### channelFlow {}
여러 코루틴에서 동시에 값을 send 할 수 있는 flow 생성

동시성이 필요한 경우 사용

```kotlin
val flow = channelFlow {
    launch { send(1) }
    launch { send(2) }
}
```

### callbackFlow {}
콜백 기반 API를 flow로 변환할 때 사용

반드시 awaitClose {}를 통해 콜백 해제 필요

```kotlin
fun flow(): Flow<Location> = callbackFlow {
    val callback = object : LocationCallback {
        override fun onLocation(loc: Location) {
            trySend(loc)
        }
    }
    
    locationManager.register(callback)
    awaitClose { locationManager.unregister(callback) }
}
```

## 5. 중간 연산자 (Intermediate Operators)
중간 연산자는 upstream Flow에 적용되어 downstream Flow를 반환

연산자 자체는 suspend 함수가 아니며 cold로 동작

### 주요 중간 연산자
- map, 각 값을 변환
- filter, 조건을 만족하는 값만 반환
- transform, 각 값을 변환 (emit 여러 번 가능)
- take(n), 앞의 n개만 가져옴
- drop(n), 앞의 n개를 건너뜀
- onEach, 각 값에 대해 side effect 수행
- debounce(ms), 지정 시간 내에 새 값이 오지 않으면 emit
- distinctUntilChanged, 연속된 중복 값 제거
- flatMapConcat, 순차적으로 여러 flow를 합쳐 하나로 만듬
- flatMapMerge, 동시에 여러 flow를 합쳐 하나로 만듬
- flatMapLatest, 새 값이 오면 이전 flow 취소 후 새 flow로 작업
- combine(other), 두 flow의 최신 값을 결합
- zip(other), 두 flow의 값을 쌍으로 결합

## 6. 종단 연산자 (Terminal Operators)
종단 연산자는 실제로 flow 수집을 시작하는 suspend 함수

### 주요 중단 연산자
- collect, 기본적인 값 수집
- collectLatest, 새 값이 오면 이전 collect 블록을 취소
- toList / toSet, 컬렉션으로 변환
- first / firstOrNull, 첫 값 하나만 수집
- single, 정확히 하나만 수집 (둘 이상 오류)
- reduce / fold, 누적 연산 (fold는 초기 값 필요)
- count, 개수 세기
- launchIn(scope), 별도 코루틴에서 collect 시작

## 7. Flow Context, flowOn
기본적으로 producer는 collect 를 호출한 코루틴의 컨텍스트에서 실행됨 - 컨텍스트 보존 (context preservation)

flow 내부에서 withContext 를 통해 컨텍스트를 바꾸려고 하면 IllegalStateException 발생

컨텍스트를 변경하려면 flowOn 사용 필요, flowOn는 upstream의 컨텍스트만 변경

## 8. 버퍼링, 백프레셔
flow는 기본적으로 순차적 -> producer가 값을 emit하면 collector가 처리를 끝낼 때까지 기다림

collector가 느리면 전체 처리 시간 길어짐

### buffer
producer와 collector를 별도 코루틴에서 병렬 실행

### conflate
collector가 처리중일 때 새 값이 오면 중간 값을 건너뛰고 최신 값만 처리

### collectLatest
새 값이 emit되면 이전 collector 블록을 취소하고 새 값으로 다시 시작

## 9. 예외 처리

### try-catch
collect 블록을 랩핑해서 예외를 잡을 수는 있음 -> 예외 투명성을 해치므로 권장되지 않음

### catch {}
upstream 에서 발생한 예외만 잡음, downstream 의 예외는 잡을 수 없음

### onCompletion {}
flow 의 에러 유무에 관계 없이 마지막에 호출됨, 파라미터로 예외를 받을 수 있음

## 10. StateFlow
StateFlow 는 현재 상태 값을 보유하는 hot observable flow

### 특징
- 초기값 필수, 생성 시 반드시 초기값을 제공
- 현재 값 보유, 현재 값에 즉시 접근 가능
- 구독 시 최신 값 전달, 새 구독자는 바로 현재 값을 받음
- 중복 제거, 동일한 값이 연속으로 set되면 emit하지 않음
- conflated, 느린 Collector는 중간 값을 놓칠 수 있음

### stateIn 연산자
flow를 StateFlow로 변환

## 11. SharedFlow
SharedFlow 는 여러 collector에게 값을 broadcast 하는 hot flow

### 특징
- 초기값 없음
- 현재 값 보유 안 함, 상태가 아닌 이벤트에 적합
- replay 설정 가능, 새 구독자에게 과거 값 N개를 재생할 수 있음
- 버퍼 정책 설정, onBufferOverflow 로 버퍼 오버플로우 동작 지정
- 절대 완료되지 않음, collect 가 정상 종료되지 않음

### shareIn 연산자
flow 를 SharedFlow 로 변환, 여러 collector가 하나의 upstream을 공유하게 만들어 리소스를 절약할 수 있음

### SharingStarted
- Eagerly, scope 시작과 동시에 upstream 수집 시작, 취소되지 않음
- Lazily, 첫 구독자가 나타나면 수집 시작, 이후 계속 유지
- WhileSubscribed(timeout), 구독자가 있을 때만 수집, 모두 사라지면 timeout 후 중단

## 14. 참고 자료
- [Kotlin 공식 문서 - Asynchronous Flow](https://kotlinlang.org/docs/flow.html)
- [API Reference - SharedFlow](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-shared-flow/)
- [API Reference - StateFlow](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-state-flow/)

## 15. 문제로 낼만 한거

### 기본
- flow 개념
- flow, suspend 함수 차이 설명
- cold flow, hot flow 차이 설명

### 세부
- collect, collectLatest 차이 설명
- map, transform 차이 설명
- StateFlow, SharedFlow 차이 설명
- stateIn, shareIn 개념 및 언제 쓰이는지
- SharingStarted - Eagerly, Lazily, WhileSubscribed 차이
- callbackFlow 언제 사용, awaitClose 필요 이유
- flow 예외 처리
- StateFlow가 같은 값을 두 번 set해도 emit되지 않는 이유, 우회하는 방법
- StateFlow로 일회성 이벤트를 처리하면 생기는 문제점
- combine, zip 차이







