# Kotlin 스코프 함수

## 한눈에 비교

| 함수 | 객체 참조 | 반환값 | 주요 용도 |
| --- | --- | --- | --- |
| `let` | `it` | 람다 결과 | null 체크, 변환 |
| `run` | `this` | 람다 결과 | 초기화 + 결과 반환 |
| `with` | `this` | 람다 결과 | 객체 멤버 다중 호출 |
| `apply` | `this` | 객체 자신 | 객체 설정/빌더 패턴 |
| `also` | `it` | 객체 자신 | 부수 효과 (로깅 등) |

반환값 기준으로 나누면

`let`, `run`, `with`는 람다 결과를 반환하고,
`apply`와 `also`는 객체 자신을 반환한다.

---

### let

객체를 `it`으로 참조하고 람다 결과를 반환한다

`?.let` 조합으로 null-safe 처리할 때 제일 많이 쓴다

널쳌 이외에도
1. 지역 변수의 스코프를 블록 안으로 제한할 때
2. 체이닝 중간에 타입을 변환할 때
   유용하다

```kotlin
val name: String? = "아딘"

val length = name?.let {
    println("나는 $it")
    it.length
}
// length == 2
```

### run

객체를 `this`로 참조하고 람다 결과를 반환한다

`with`랑 하는 일이 비슷하지만 확장 함수로 동작해서 `?.run { }` 처럼 안전 호출이 가능하다

객체를 초기화함과 동시에 결과를 뽑아야 할 때 쓴다.

```kotlin
val result = "mya".run {
    println(length) // length == 3
    uppercase()
}
// result == "MYA"
```

```kotlin
val userName = intent.extras?.run {
    getString("USER_NAME") 
}
```

```kotlin
val formattedDate = SimpleDateFormat("yyyy-MM-dd").run {
    timeZone = TimeZone.getTimeZone("Asia/Seoul")
    format(Date()) 
}
// formattedDate == "2026-04-13" 
```

### with

객체를 `this`로 참조하고 람다 결과를 반환한다

확장 함수가 아니라 `with(객체) { }` 형태로 쓴다

`run`과 동작은 같지만 null이 아닌 게 확실한 객체의 멤버를 여러 번 호출해서 그 객체의 속성을 **우르르 몰아서 세팅**할 때 가독성 굿

```kotlin
val result = with(StringBuilder()) {
    append("아딘")
    append(" 먀우")
    toString()
}
// result == "아딘 먀우"
```

컴포즈에서 주로.. uiState

```kotlin
@Composable
fun DetailScreen(uiState: DetailUiState) {
    with(uiState) {
        if (isLoading) {
            CircularProgressIndicator()
        } else {
            Column {
                Text(text = title)       // uiState.title
                Text(text = description) // uiState.description
                
                if (isBookmarked) {      // uiState.isBookmarked
                    Icon(imageVector = Icons.Default.Favorite, contentDescription = null)
                }
            }
        }
    }
}
```

### apply

객체를 `this`로 참조하고 객체 자신을 반환한다

프로퍼티를 `this` 없이 바로 설정할 수 있어서 객체 초기화나 빌더 패턴에 자주 쓰인다.

```kotlin
val person = Person().apply {
    name = "기마딘"
    age = 5
    home = "하남"
}
```

```kotlin
@Composable
fun CustomWebView(url: String) {
    AndroidView(
        factory = { context ->
            WebView(context).apply { // 뷰를 생성하자마자 apply로 속성을 세팅해줌
                settings.javaScriptEnabled = true
                settings.domStorageEnabled = true
                webViewClient = WebViewClient()
            }
        },
        update = { webView ->
            webView.loadUrl(url)
        }
    )
}
```

### also

객체를 `it`으로 참조하고 객체 자신을 반환한다

`apply`와 반환값은 같지만 `it`으로 참조하기 때문에 객체 내부를 건드리기보다 외부에서 부수 작업을 할 때 사용

체이닝 흐름을 끊지 않고, 로깅이나 유효성 검사를 끼워넣을 때 유용하다

```kotlin
fun getUserProfile() {
    apiService.fetchUser()
        .also { Log.d("NetworkLog", "서버 응답 결과: ${it.name}") } 
        .let { uiState.value = it } 
}
```

---

## 정리하면

```
null 처리하거나 값 변환        → let
프로퍼티 초기화               → apply
체이닝 중간에 로깅/검증        → also
결과값이 필요한 블록           → run
멤버 여러 개 한 번에 호출      → with
```

---

## Q. 스코프 함수란 뭔가요?

스코프 함수는 객체의 컨텍스트 안에서 코드 블록을 실행할 수 있게 해주는 표준 라이브러리 함수들입니다

let, run, with, apply, also 다섯 가지가 있으며
객체를 this로 참조하는지 it으로 참조하는지,
람다 결과를 반환하느냐 객체 자신을 반환하느냐에 따라 구분됩니다

이 스코프 함수를 적절히 활용하면 불필요한 변수 선언을 줄여 코드의 가독성을 크게 높일 수 있고,
안전한 Null 처리나 객체 초기화를 매우 간결하게 작성할 수 있습니다.

---

## Q. **`let`과 `also`의 차이가 뭔가요?**

둘 다 객체를 `it`으로 참조하지만 반환값이 다릅니다

`let`은 람다의 마지막 표현식을 반환하고, `also`는 객체 자신을 반환합니다

그래서 `let`은 값을 변환하거나 null 처리할 때 쓰고,
`also`는 체이닝 흐름을 유지하면서 로깅이나 검증 같은 부수 작업을 끼워넣을 때 사용합니다

---

## Q. **`apply`와 `also`는 둘 다 객체 자신을 반환하는데 어떻게 구분하나요?**

반환값은 같지만 객체 참조 방식이 다릅니다.

`apply`는 `this`로 참조, `also`는 `it`으로 참조합니다.

그래서 `apply`는 객체 내부를 설정할 때, `also`는 객체를 그대로 두고 외부에서 부수 작업을 할 때 씁니다.

---

## Q. **`run`과 `with`는 어떤 차이가 있나요?**

둘 다 `this`로 참조하고 람다 결과를 반환합니다

차이는 호출 방식인데, `run`은 확장 함수라서 `객체.run { }` 형태로 쓰고 `?.run { }` 처럼 안전 호출도 할 수 있습니다

`with`는 일반 함수라서 `with(객체) { }` 형태로 씁니다

null 가능성이 있는 객체라면 `run`, null 체크가 끝난 객체를 다룰 때는 둘 다 써도 무방합니다

---

## **Q. 스코프 함수 중첩**

가능하긴 하지만 권장하지 않고 선호하진 않습니다

중첩되면 `it`이나 `this`가 어느 객체를 가리키는지 헷갈리기 쉽고,
가독성을 위해 사용하는데 가독성이 너무 떨어집니다

특히 `it`을 쓰는 함수를 중첩하면 안쪽 `it`이 바깥 `it`을 가려버려서 의도치 않은 버그가 생길 수 있기도 합니다

중첩이 불가피하다면 람다 파라미터에 명시적으로 이름을 붙여야 합니다

---

## **Q. 스코프 함수가 성능에 영향을 주지는 않는지?**

스코프 함수는 람다를 인자로 받기 때문에 원래라면 객체 생성 오버헤드가 발생해야 하지만,
코틀린 컴파일러가 인라인 처리를 해줘서 실제로는 대부분 오버헤드가 없다

다만, inline 함수의 특성상 무분별하게 남발하거나 깊게 중첩하면
컴파일된 바이트코드의 크기가 커지는 부작용이 있을 수 있습니다

성능이 극도로 중요한 루프 안에서 남발하는 건 피하는 게 좋지만, 일반적인 상황에서는 가독성을 위해 적극적으로 활용하는 것 같습니다