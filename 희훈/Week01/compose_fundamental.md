# Jetpack Compose 기본 동작

## 1. 개요

Jetpack Compose는 안드로이드의 선언형 UI로, 기존의 XML 기반 명령형 View 시스템을 대체하기 위해 개발된 것

Kotlin 코드를 통해 UI를 작성하고, 상태가 변경되면 프레임워크가 다시 그려야할 부분을 다시 그림

핵심 원리
- UI는 상태: 상태가 바뀌면 다시 실행되어 UI가 갱신됨
- Composable 함수: @Composable 어노테이션이 붙은 Kotlin function이 기본 단위
- 단방향 데이터 흐름: 상태는 위에서 아래로 전파되고, 이벤트는 아래서 위로 전파됨
- Smart Recomposition: 변경된 최소한의 부분만 갱신함

## 2. 선언형 UI

전체를 다시 그려야 함 - 상태가 변경되면 전체 UI를 처음부터 다시 그림

최적화 - Compose 런타임이 변경된 부분만 다시 갱신함

UI와 상태간의 불일치 해소 - 상태와 UI는 항상 같은 인풋에 대한 결과가 같기 때문에 View 시스템에서 발생했던 상태 불일치 미발생

### 명령형

UI의 변경을 명령하여 View를 갱신하는 방식

View를 찾거나, 속성을 변경하는 등 모든 변경 절차를 개발자가 직접 코드를 짜야함

```kotlin
val textView = findViewById<TextView>(R.id.test)
textView.text = "Test"
```

### 선언형 UI

현재 상태에 대해 UI가 어떤식으로 보여져야 하는지 기술하는 방식

```kotlin
@Composable
fun TempTextView() {
    Text(text = "Test")
}
```

## 3. Composable 함수

### 기본 개념

@Composable 어노테이션이 붙은 함수는 일반적인 Kotlin 코드와 다르게 동작

Compose 컴파일러 플러그인이 어노테이션이 붙은 함수를 변환하여 Composer 객체를 전달 받게 함

```kotlin
@Composable
fun TempTextView() {
    Text(text = "Test")
}

// compiler
fun TempTextView(
    $composer: Composer<*>,
) {
    Text(
        text = "Test",
        $composer,
    )
}
```

- Composable 함수의 특성
    - 반환값이 Unit - 기존처럼 View를 return 하지 않고 composer에 emit함
    - 호출 순서가 보장되지 않음 - 최적화를 위해 컴파일러가 Composable 함수의 실행 순서 변경 가능
    - 병렬 실행 가능 - 서로 독립적인 Composable 함수는 병렬 실행 가능
    - 빈번하게 재실행 - 프레임마다 실행될 수 있음
    - 멱등성 - 같은 값으로 여러 번 호출해도 동일한 결과를 내야함

## 4. Compose의 세 가지 렌더링 단계

### 단계 1: Composition

Compose 런타임이 Composable 함수를 실행하고, UI 노드 트리 구성

화면에 그리지 않고, 메모리에 설계도를 만드는 작업

결과적으로 LayoutNode로 구성된 UI 노드 트리 구성

```kotlin
@Composable
fun TempTextView() {
    // UI 트리에 Column, Text 노드가 등록됨
    Column {
        Text(text = "Test")
    }
}
```

### 단계 2: Layout

Composition에 생성된 UI 노드 트리를 돌면서 각 노드의 크기와 위치 결정

1. 부모 노드가 자식 노드에게 크기 측정 요청 (measure)
2. 자식들의 크기를 기준으로 자신의 크기를 결정
3. 자식 노드를 2차원 좌표에 배치

각 노드는 크기를 한번만 확인하기 때문에 O(n)으로 과정이 완료됨

이는 기존 View 시스템과 비교하면 성능상 큰 이점을 지님

### 단계 3: Drawing

UI 트리를 돌면서 각 노드가 Canvas 위에 그려짐

실제로 렌더링되는 단계

### 단계 최적화

세 단계는 선형적으로 실행되지만, Compose는 필요한 단계만 실행하기도 함

- UI 구조를 변경 -> 세 단계 모두 실행
- 크기, 위치 변경 -> 2, 3단계 실행
- 색상, 투명도 변경 -> 3단계 실행

## 5. Composition과 UI 트리

앱이 처음 실행되면 Compose는 모든 Composable 함수를 실행하여 UI 트리 구축

구축된 트리는 LayoutNode의 계층 구조로 각 UI 요소를 메모리에 표현한 것

### UI 트리의 구성 요소

- LayoutNode: 각 UI의 요소를 나타낸 것, UI의 정보를 가지고 있음
- Modifier Node: Modifier 체인에 의해 생성되는 노드
- Composition: 트리 전체를 관리하는 객체, Slot Table을 가짐, Recomposer에 의해 갱신

실제 Android View를 생성하지 않고 메모리에 노드를 만들고 렌더링때 Canvas에 직접 그림

## 6. Recomposition(재구성)

상태가 변경되었을 때 해당 상태를 읽고 있는 Composable 함수만 재실행하여 UI 트리를 갱신

전체 UI를 다시 그리지 않고 필요한 부분만 재실행

### 핵심 특성

- Composable 함수 단위로 발생
- 스킵 가능: 모든 인풋이 이전과 동일하면 재실행을 건너뛸 수 있음
- 낙관적(Optimistic): 재구성 도중 상태가 변경되면 진행중인 재구성을 취소하고 다시 시작
- 비순차적 실행: 실행 순서가 보장되지 않고 병렬 실행 가능

## 7. Slot Table

```kotlin
/**
 * 이거 너무 어렵다
 * 공부하면서 작성하면서도 너무 헷갈림
 */
```

Compose 런타임의 핵심 내부 데이터 구조, Composition의 모든 정보를 순차적으로 저장하면 선형 배열

기존 View는 트리 구조의 View Hierarchy, Compose는 Slot Table 평면 데이터 구조를 갖음

### 저장되는 정보

- Composable 함수의 시작/종료 마커
- remember로 저장된 값
- Composable input 값
- 상태 객체(MutableState) 참조
- key 값으로 지정된 식별자

### Gap Buffer 구조

텍스트 에디터에서 커서 위치 근처에 삽입/삭제를 효율적으로 처리하는 원리

- 삽입/삭제: Gap 위치 근처에서는 상수 시간 (O(1))
- Gap 이동: 데이터의 복사가 필요, Recomposition에서는 순차적으로 접근하기 때문에 효율적

### 위치 기억법 (Positional Memoization)

소스 코드의 호출 위치를 기준으로 Composable의 동일성 판단

같은 위치에서 호출된 Composable은 이전 Composition 데이터를 재사용 가능

e.g. List

key를 사용하지 않으면, 리스트 항목의 CRUD, Sort 때 의도하지 않은 동작이 발생할 수 있음

## 8. State의 기본 원리

상태는 시간에 따라 변경될 수 있고 UI에 직접 영향을 미치는 데이터

상태가 바뀌면 해당 상태를 읽고 있는 Composable이 Recomposition 됨

### mutableStateOf와 remember

```kotlin
@Composable
fun Temp() {
    // Recomposition시 0으로 초기화됨
    var temp = mutableStateOf(0)
    
    // remember로 감싸서 Composition에 저장
    var count by remember { mutableStateOf(0) }
    
    // 화면 회전 같은 구성 변경에도 살아남음
    var count by rememberSaveable { mutableStateOf(0) }
}
```

- mutableStateOf: Compose가 관찰할 수 있는 객체를 생성
- remember: 값을 Slot Table에 저장, Recomposition시 유지됨
- rememberSaveable: remember의 역할과 Bundle에 저장

### State Hoisting

상태를 Composable 내부가 아니라 호출자에게 위임하는 패턴

Composable 함수를 Stateless 하게 만들어서 재사용성 향상, 테스트 용이

## 9. Snapshot System

Compose 런타임이 상태 변경을 감지하고 Recomposition을 트리거하는 매커니즘

특정 시점의 상태를 Snapshot 하여 관리

### 동작 원리

- 상태 읽기 추적: Composable 함수 실행 중 MutableState의 value를 읽으면 상태 의존 기록
- 상태 쓰기 감지: 상태가 변경되면 시스템이 이를 감지
- 무효화: 해당 상태를 읽고 있던 Composable을 무효 상태로 표시
- 재구성 스케쥴링: 무효화된 Composable만 재구성 예약

### 원자성

메인 스레드에서 연속으로 여러 상태를 변경하면 해당 변경들은 하나의 Recomposition에 반영됨

## 10. Composable의 생명주기

1. Composition 진입: Composable이 처음 실행되고 UI 트리에 추가됨
2. Recomposition: 상태 변화에 따라 재실행됨
3. Composition 이탈: UI 트리에서 제거됨

### Composition 진입/이탈의 조건

```kotlin
@Composable
fun TempTextView(showTemp: Boolean) {
    Column {
        Text("Test")  // 항상 Composition에 존재
        
        if (showTemp) {
            // showTemp가 true → Composition 진입
            // showTemp가 false → Composition 이탈
            TempScreen()
        }
    }
}
```

showTemp가 false가 되면 TempScreen는 Composition에서 제거됨

내부의 remember 값 등 모든 것이 dispose 되며, true로 변경될 시 처음부터 시작됨

## 11. SideEffect API

Composable 함수 scope 밖에서 발생하는 상태 변경

Composable은 언제든 재실행, 취소 될 수 있기 때문에 API를 사용하여 제어해야함

### 주요 API

- LaunchedEffect: 코루틴을 생명 주기에 맞게 실행
- DisposableEffect: 리소스 등록/해제
- SideEffect: 매 Recomposition 성공 후 실행
- rememberCoroutineScope: 이벤트 핸들러에서 코루틴이 필요할 때 사용
- derivedStateOf: remember 등으로 파생된 상태 최적화
- snapshotFlow: Compose State를 Flow로 전환

## 12. Modifier 체이닝

### Modifier 정의

Composable의 외형과 동작을 변경하는 도구

크기, 패딩, 배경 등 다양한 속성을 Composable에 추가

### 체이닝 순서의 중요성

Modifier는 체인 순서로 적용됨 -> 순서에 따라 결과가 정해짐

```kotlin
Modifier
    .padding(16.dp)
    .background(Color.Black)

Modifier
    .background(Color.Black)
    .padding(16.dp)
```

두 Modifier의 결과값이 다름

### UI 트리

Modifier 체인은 내부적으로 Wrapper 노드가 되어서 LayoutNode를 랩핑

## 13. Smart Recomposition

Compose는 Recomposition시 모든 Composable을 재실행하지 않음

입풋이 변경되지 않은 Composable은 스킵함

### 스킵 조건

1. 모든 파라미터는 Stable 해야함 -> (Strong Skipping Mode 이거 때문에 아닐 수 있음)
2. 모든 파라미터 값이 이전 Composition과 동일해야함

### Stability(안정성)

Compose 컴파일러는 타입 안정성을 분석하여 Recomposition 최적화

- Stable
    - 원시 타입
    - data class 모든 프로퍼티 val이고 Stable
    - 람다: 캡쳐된 변수가 Stable한 경우

- Unstable
  - collection 인터페이스 - mutable일 수 있음
  - var 프로퍼티를 가진 경우

## 14. 참고 자료

- [Android 공식 Compose Phases](https://developer.android.com/develop/ui/compose/phases)
- [Android 공식 Compose Lifecycle](https://developer.android.com/develop/ui/compose/lifecycle)
- [Android 공식 Compose State](https://developer.android.com/develop/ui/compose/state)
- [Android 공식 Compose SideEffect](https://developer.android.com/develop/ui/compose/side-effects)
- [Android 공식 Compose Phases performance](https://developer.android.com/develop/ui/compose/performance/phases)
- [Android 공식 Compose Mental-Model](https://developer.android.com/develop/ui/compose/mental-model)
- Jetpack Compose Internals 한본판
