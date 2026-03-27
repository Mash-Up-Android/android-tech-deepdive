# 2주차 - 컴포즈 기본 동작

---

## Jetpack Compose

Compose는 상태(State)를 기반으로 UI를 선언하고, 상태 변경 시 State를 읽은 Composable만 재실행(Recomposition)하는 UI 프레임워크이다.

> UI = f(state)
상태가 변경되면 UI는 자동으로 갱신됨


## 전체 동작 흐름

1. @Composable 함수 작성
2. Compose Compiler (Kotlin IR 변환)
3. Slot Table 기반 UI 구조 생성
4. State 변경 발생
5. Recomposition (변경된 부분만 재실행)
6. UI 업데이트

---

## 기존 XML 방식과의 차이

### View(명령형): 어떻게 바꿀지 직접 제어

```kotlin
textView.text = "Hello"
```

- UI 변경을 직접 명령
- 상태와 UI가 분리됨

### Compose(선언형): 어떤 UI인지 선언만

```kotlin
Text("Hello")
```

- UI는 상태의 결과
- 상태만 변경하면 UI 자동 반영

---

## 핵심 메커니즘

### State 기반 UI

```kotlin
Text("Count: $count")
```

- UI는 상태를 기반으로 생성됨
- 상태 변경 시 UI 자동 업데이트

### State Read Tracking

- Composable 실행 시 어떤 State를 읽었는지 기록
- 상태 변경 시 해당 Composable만 invalidation

### Recomposition

- 전체 UI 재실행이 아니라 변경된 State를 읽은 Composable만 재실행

### Slot Table

Composable 실행 결과와 상태를 저장하는 내부 자료구조

- UI 구조 저장
- remember 값 저장
- 이전 파라미터 값 저장
- Recomposition 시 재사용

### Skipping (성능 최적화)

invalidation 이후에도 실제 변경이 없다고 판단되면 Composable 실행을 건너뛴다.

조건

- 파라미터 변경 없음
- Stable 타입
- 영향 있는 State 변경 없음

---

## Compose Compiler 역할

@Composable 어노테이션을 붙이면 컴파일 타임에 함수 시그니처가 변경된다.

```kotlin
// before
@Composable
fun Text(text: String)

// after
fun Text(text: String, composer: Composer, changed: Int)
```

- Composer 파라미터 추가
- changed flag로 skip 여부 판단
- restartable / skippable 함수로 변환

---

## 참고 자료

https://developer.android.com/develop/ui/compose/state?hl=ko

https://velog.io/@sinabro0209/Compose%EB%8A%94-%EC%96%B4%EB%96%BB%EA%B2%8C-State-%EB%B3%80%ED%99%94%EB%A5%BC-%EA%B0%90%EC%A7%80%ED%95%A0%EA%B9%8C-Snapshot%EA%B3%BC-Recomposition%EC%9D%98-%EB%82%B4%EB%B6%80-%EC%9B%90%EB%A6%AC
