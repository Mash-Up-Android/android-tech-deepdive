## 📌 MVI 패턴이란 무엇인가요?

MVI는 Model-View-Intent 패턴으로, 단방향 데이터 흐름(Unidirectional Data Flow)을 기반으로 설계된 아키텍처 패턴입니다.

## 📌 단방향 데이터 흐름이란 무엇인가요?

단방향 데이터 흐름이란 데이터가 항상 한 방향으로만 흐르는 아키텍처 패턴입니다

ui에서 이벤트가 발생하면 viewModel을 통해 state가 변경되고.. state가 변경되면 ui에 반영되고...

다시 또 ui에서 이벤트가 발생하면... (반복)

이렇게 한 방향으로만 흐르면 상태는 viewModel이 관리하고

ui는 상태를 구독해서 화면에 뿌리는 역할만 담당하게 됩니다.

### 🤔 그러면 MVI는 단방향 데이터 흐름, MVVM은 양방향 데이터 흐름이라고 보면 되는건가?

MVVM도 단방향 데이터 흐름이 될 수 있음!

단방향 데이터 흐름은 MVI만의 전유물이 아님

정확히 하자면 MVI는 UDF를 '강제'하는 구조이고

MVVM은 UDF를 '선택적으로' 구현 할 수 있음

```
// ❌ View가 ViewModel 상태를 직접 수정 → UDF 위반
viewModel.uiState.value.email = "test@test.com"

// ❌ 양방향 바인딩으로 View ↔ Model 직접 동기화
// DataBinding의 @={} 양방향 바인딩
```

예를 들어 위 코드처럼 view가 직접 상태를 변경한다거나, 양방향 바인딩을 사용하면 UDF가 깨져버리는 것임

```
// ✅ View는 이벤트만 전달, 상태는 ViewModel이 관리
viewModel.onEmailChanged("test@test.com")
```
근데 이렇게만 사용했다? 그러면 UDF를 지킨거지

```
// Intent로만 이벤트 전달 가능 → 직접 상태 수정 불가능
sealed class LoginIntent {
    data class EmailChanged(val email: String) : LoginIntent()
    object LoginClicked : LoginIntent()
}

// View는 Intent만 보낼 수 있음
viewModel.processIntent(LoginIntent.EmailChanged("test@test.com"))
```

반면 MVI는 Intent로만 이벤트를 전달할 수 있다보니까

view가 직접 상태를 수정할 수 없고, UDF를 아키텍처 차원에서 강제할 수 있는것임

## 📌 MVI 패턴을 사용하면서 느낀 단점은 없었나요?

state, Intent, Effect를 모두 정의해야 하다보니 단순한 화면에도 보일러플레이트가 많이 생기는 단점이 있었습니다.

그래서 프로젝트에 따라 intent를 사용하지 않는 대신

view가 viewModel의 상태를 직접 수정하지 않도록 캡슐화하고, 컨벤션을 두는 등 방식으로 보일러플레이트를 줄이는 방식을 택하곤 했습니다.


그리고 화면이 복잡해질수록 state 클래스가 거대해지는 문제도 있었습니다.

이는 state를 논리적인 단위로 분리하여 해결할 수 있었습니다.


마지막으로는 다른 패턴에 비해 러닝커브가 높다는 단점이 있었습니다.

이는 orbit과 같은 mvi 패턴 라이브러리를 통해 학습해 해결할 수 있었습니다.

### 🤔 Intent를 사용하지 않으면 MVI라고 할 수 없는건가?

view가 viewModel의 함수를 호출해서 uiState를 변경하든

view가 intent를 통해 uiState를 변경하든

둘 다 단방향 흐름이라는 MVI 정신은 지키는거임!

후자가 더 MVI 스럽게 보이는 것일 뿐, 본질은 같음
