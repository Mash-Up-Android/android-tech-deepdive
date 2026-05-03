## 📌 코루틴이란?

‘중간에 멈췄다가 재개할 수 있는 함수’ 입니다.

일반 함수는 한 번 호출하면 스코프가 끝날때까지 모든 실행문이 실행되잖아요.

근데 코루틴은 중간에 하던 작업을 멈추고 잠깐 다른 작업을 하러갔다가 돌아와서 재개할 수 있습니다.

이러한 특징 덕분에

흔히들’ 코루틴을 경량 스레드’라고도 많이 표현하더라고요.

여러 작업들을 동시에 다루는 것처럼 보인다 그렇게도 많이 정의해서 부르곤 합니다.

## 📌 suspend 함수란?

코루틴 안에서 suspend 함수를 호출하면 코루틴의 흐름이 멈추게됩니다.

suspend 함수 작업이 끝나면 다시 이어서 코루틴 흐름이 진행돼요.

즉, suspend 함수는 코루틴 흐름을 멈추게 할 수 있는 함수

일시중단 할 수 있는 함수입니다.

## 📌 dispatcher란?

코루틴이 어느 스레드에서 실행될지 결정하는 놈입니다.

Dispatchers.Main // UI 작업

Dispatchers.IO // 네트워크, DB (최대 64개 스레드) 

Dispatchers.Default // CPU 집약적 작업 (CPU 코어 수만큼) 

## 📌 launch와 async의 차이점은?

launch는 job을 리턴

asnyc는 Deffered를 리턴

launch는 fire-and-forget, 그래서 결과가 필요없는 작업

async는 await로 결과 수신, 그래서 결과가 필요한 작업

## 📌 Deffered란?

Deffered는 Job을 확장하는 인터페이스입니다. 실제로 코드를 까보면 job을 구현하고 있고..

job에다가 +a로 결과값을 리턴해주는 인터페이스라고 생각하면 됩니다.

job은 결과값이 없잖아요. Deffered는 결과값이 있습니다.

한마디로 Deferred는 결과값을 수신하는 비동기 작업이다.. 라고 할 수 있습니다.

## 📌 withContext란?

withContext는 코루틴의 실행 컨텍스트(Dispatcher)를 바꿔서 코드 블록을 실행하는 함수입니다.

## 📌 코루틴에서 메모리 누수를 방지하는 방법은?

viewModelScope / lifecycleScope 사용으로 생명주기 자동 관리

repeatOnLifecycle 사용

GlobalScope 사용 지양

Job을 저장해두고 필요 시 cancel() 명시적 호출

## 📌 코루틴에서 동시성 문제가 발생할 수 있는 상황은 어떤 것이 있나요?

한 코루틴 내에서는 문제가 없겠으나..

여러 코루틴 스코프를 열어 하나의 자원에 접근하면 동시성 문제가 발생할 수 있습니다.

이런 경우는 atomic 타입을 사용하거나 Mutex를 사용하는 등 방법을 사용해서

여러곳에서 하나의 자원을 차지하는 일이 없도록 방지해야 합니다.

## 📌 yield란?

스레드의 선점권을 넘길때 사용하는 키워드입니다.

예를 들어 굉장히 무거운 작업을 한다면 그 작업이 끝날때까지 스레드가 계속 묶이게 되잖아요.

그래서 작업 중간 중간에 잠시 선점권을 넘겨 다른 작업을 할 수 있도록 할 때 사용합니다.

저 같은 경우는 어떠한 작업중에 잠시 메인스레드의 선점권을 넘겨주어

데이터가 화면에 실시간으로 반영되는 과정을 표현하기 위해 사용한 적이 있습니다.

## 📌 globalScope란?

앱의 생명주기 전체와 동일한 범위를 가지는 코루틴 스코프로, 특정 컴포넌트(Activity, Fragment, ViewModel)에 묶이지 않고 앱이 살아있는 동안 계속 실행됩니다.

메모리 누수를 발생시킬 수 있어서 웬만하면.. 사용하지 않습니다.

## 📌 CoroutineScope의 종류와 특징에 대해 설명해주세요.

globalScope는 앞에서 설명했으니 패스 ㅎ

viewModelScope는 viewModel의 생명주기를 따르는 스코프

lifecycleScope는 activity/fragment의 생명주기를 따르는 스코프

rememberCoroutineScope는 composable의 생명주기를 따르는 스코프

CoroutineScope는 커스텀 스코프. 직접 생명주기를 조절할 수 있음

## 📌 CancellationException이란?

CancellationException은 코루틴이 취소되면 발생하는 Exception 입니다.

Kotlin 코루틴에서 취소가 Exception을 통해 구현됩니다.

일반 Exception은 발생하면 부모에게 전파되지만

CancellationException은 부모 코루틴에게 전파되지 않습니다.

### 🤔 CancellationException을 try~catch로 감싸서 무시하면 안되는 이유

```
// ❌ 잘못된 코드 - 취소가 동작하지 않음
launch {
    try {
        delay(1000)
    } catch (e: Exception) { // CancellationException도 잡힘
        // 아무것도 안 함 → 코루틴이 취소되지 않음!
    }
    println("취소됐는데 계속 실행됨") // 실행돼버림
}
```

CancellationException이 전파되어야 코루틴이 취소되는데

위 코드처럼 catch한 다음 다시 throw 하지 않으면 코루틴이 계속 살아있습니다.

그렇게 되면 의도한 생명주기를 벗어나서 코루틴이 계속 살아있을수도 있습니다.

```
launch {
    try {
        // 무언가 작업...
    } catch (e: CancellationException) {
        throw e // 반드시 재throw
    } catch (e: Exception) {
        // 다른 예외만 처리
    }
}
```

그래서 exception이 발생했을 때 에러를 발생시키지 않고 예외 처리를 해주고 싶다면

CancellationException는 다시 throw 해주고

일반 exception만 따로 처리해주는것이 필요합니다.

### 🤔 자식 스코프에서 일반 exception이 발생하면 생기는 일

1. 자식 스코프에서 exception 발생
2. 자식 스코프의 작업이 실패로 바뀌면서 종료
3. 부모 스코프로 exception 전파 (CancellationException 였으면 전파 안됨! 일반 Exception 이니까 전파되는거임)
4. 부모 스코프가 다른 자식 스코프에게 CancellationException을 전파해서 취소시킴 (cancel()을 호출해서 자식들이 CancellationException을 수신)
5. 자식 스코프들 정리되면 부모 스코프도 실패로 바뀌면서 종료
6. ExceptionHandler 실행

### 🤔 CancellationException을 의도적으로 무시해야 하는 케이스도 있을까?

시간 초과시 예외를 발생시키는 withTimeoutOrNull이라는 함수는 내부적으로 다음과 같이 생겼습니다. (대략적으로)

```
try {
    withTimeout(1000L) { fetchData() }
} catch (e: TimeoutCancellationException) {
    null // 의도적으로 무시
}
```

TimeoutCancellationException가 CancellationException의 하위 클래스이기 때문에

시간초과가 되었을 때 스코프를 끝내지 않고 null을 리턴해 처리해주기 위해

catch문으로 잡아 위와 같이 처리해주는 것입니다.

```
// withTimeout → 시간 초과시 예외 발생
val result = withTimeoutOrNull(1000L) {
    fetchData() // 1초 안에 못 끝내면
} ?: "기본값" // CancellationException 무시하고 null 반환
```

사용할때는 이렇게 사용함!

## 📌 Job과 SupervisorJob의 차이는?

job은 자식 코루틴이 실패하면 부모와 다른 자식 모두 취소됨

SupervisorJob은 자식 코루틴이 실패해도 다른 자식에게 영향없음

### 🤔 내가 사용해본 경험

여러 개의 이미지를 동시에 업로드하는 기능을 구현한 적이 있는데

job을 사용하면 이미지 중 하나만 업로드 실패해도 나머지도 다 실패로 처리되니까

supervisorJob을 사용해서 해당 이미지만 업로드 실패로 처리하기 위해 사용했었음

### 🤔 CancellationException은 다시 throw하고 일반 Exception은 별도로 처리해주면 SupervisorJob가 필요없지 않나?

```
val scope = CoroutineScope(Job())

scope.launch {
    launch {
        try {
            throw Exception("실패")  // ← catch로 잡음
        } catch (e: Exception) {
            if (e is CancellationException) throw e
            println("잡았다!")  // ← 여기서 처리
        }
        // 여기까지는 OK
    }
    launch {
        delay(1000)
        println("나는 살아있나?")  // ✅ 이 경우엔 살아있음
    }
}
```

단순히 launch를 사용하는 케이스에서는 맞긴한데...

```
val scope = CoroutineScope(Job())

scope.launch {
    val deferred = async {
        throw Exception("async 실패")
    }
    
    try {
        deferred.await()  // ← 여기서 예외가 터짐
    } catch (e: Exception) {
        println("잡았다!")
    }
    // 😱 catch 했는데도 부모 Job이 취소됨
}
```

문제는 async랑 await을 사용했을때임

await 사용하는곳만 try~catch로 감싼다고 해결되는 문제가아님

왜냐면 awit하는 시점이 아니라 job에 exception을 저장하기 때문임

그래서 위와 같은 케이스에서는 catch랑 무관하게 부모로 전파되어 코루틴이 취소될거임..

이렇게 되면 전반적으로 모든 스코프가 다 죽기때문에 구조적으로 취약함.

이런일 발생하면 누가 책임져줄거여? 개발자가 다 책임지는거여.

근데 SupervisorJob은 구조적으로 exception이 발생해도 부모로 전파되는 경로 자체가 끊겨있음

그래서 안전한거임!

즉, 단순한 케이스에선 동작하지만, async나 중첩 구조에서 쉽게 무너지고 실수하기 쉽고

SupervisorJob은 그 보호를 구조적으로 보장해주는 것입니당.
