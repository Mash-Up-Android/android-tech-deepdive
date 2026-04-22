## 📌 kotlin에서 open class와 일반 class의 차이점은?

Kotlin의 모든 클래스는 기본적으로 final입니다. 즉, 상속이 불가능합니다.

상속을 허용하려면 open 키워드를 명시해야 합니다. 이는 Java와 반대 방향의 설계 철학으로, 불필요한 상속을 막아 안전한 코드를 유도합니다.

```
// 상속 불가 (기본)
class Animal

// 상속 가능
open class Animal
class Dog : Animal()
```

## 📌 data class란 무엇인가요?

일반 클래스에 +a로 주로 사용하는 함수들을 미리 구현해둔 클래스입니다.

데이터를 담는 클래스로 쓰이고 그러다보니 데이터 관련된 함수들이 구현되어 있습니다.

ex) 데이터 값 끼리의 비교라거나.. 데이터 값 복사라거나...

## 📌 data class는 왜 상속이 불가능 한가요?

앞서 말했듯 data class는 미리 구현되어 있는 메서드들이 몇개 있는데요

예를 들면 equals !

```
data class Animal(val name: String)

// 자식 클래스가 프로퍼티 추가
class Dog(name: String, val age: Int) : Animal(name)

val d1 = Dog("뽀삐", 3)
val d2 = Dog("뽀삐", 5)

d1 == d2 // true ??? 😱
```

위 예시에서 Dog는 equals()를 따로 안 만들었으니 부모인 Animal의 equals()를 그대로 씁니다.

그런데 Animal의 equals()는 name만 비교하도록 만들어져 있어서, age가 달라도 같다고 판단해버립니다.

말 그대로 Dog판인 상황인거죠.

equals 말고 다른 함수도 이와 비슷한 문제들이 있어서 Kotlin에서는 data class를 상속하지 못하게 막은 것입니다.

## 📌 그럼 data class의 상속이 필요한 경우는 어떻게 하죠? 그런 케이스가 있을수도 있지 않을까요?

그럴때는 sealed class와 data class를 같이 사용하면 됩니다 찡긋 😉

## 📌 sealed class란 무엇이고 언제 사용하나요?

sealed class는 상속 계층을 제한하는 클래스입니다.

when문과 함께 사용하면 모든 케이스를 컴파일 타임에 검사할 수 있다는 장점이 있습니다.

주로 uiState를 계층구조로 표현할때나 서버에서 받는 response를 타입별로 컨버팅하는 상황 등에서 사용하곤 했습니다.

## 📌 sealed class와 sealed interface의 차이는 무엇인가요?

ealed class는 단일 상속만 가능하지만, sealed interface는 다중 구현이 가능합니다.

하나의 클래스가 여러 sealed interface를 구현할 수 있어 더 유연한 타입 계층 설계가 가능합니다.

```
sealed interface Shape
sealed interface Colorable

data class RedCircle(val r: Double) : Shape, Colorable
data class BlueRect(val w: Double) : Shape, Colorable
```

그리고 부모로부터 상속받아야 하는 값이 없으면 웬만하면 sealed interface를 사용하는편입니다.

sealed class로도 할 수 있긴 하지만... 굳이?

```
// 굳이 sealed class를?
sealed class Menu {
    data class Settings(val title: String) : Menu()
    data class UserInfo(val title: String) : Menu()
}

// sealed interface를 사용함
sealed interface Menu {
    data class Settings(val title: String) : Menu
    data class UserInfo(val title: String) : Menu
}
```

### 🤔 내가 sealed class + when문을 사용할 때 웬만하면 else를 사용하지 않는 이유

```
    fun handleCardState(card: Card?) {
        when (card) {
            Card.Activate -> // draw activate card view
            Card.Deactivate,
            null -> // set view visible as false
        }
    }
```

카드가 활성화 상태면 뷰를 그리고

비활성화 상태이거나 null 값이 들어오면 뷰를 감추는 로직이 있다고 쳤을때

```
fun handleCardState(card: Card?) {
        when (card) {
            Card.Activate -> // draw activate card view
            else -> // set view visible as false
        }
    }
```

이렇게 else로 합쳐버리고 싶을수도 있습니다.

어차피 동작도 똑같고 보기에 이게 더 간단하니까요

```
sealed interface Card {
    data object Activate : Card
    data object Deactivate : Card
    data object Suspended : Card
}
```

근데 훗날 Suspended(카드 일시정지) 상태가 추가되었고 suspended 케이스일때 그려야 하는 뷰가 있다고 가정하겠습니다.

when문에 else를 사용한 경우는 어떠한 경고도 없이 빌드가 잘 될겁니다.

-> 프로젝트 내에 Card 객체를 분기치는 곳을 전부 직접 찾아 수정해주어야 하고, 수정을 못하고 놓칠 수 있다는 위험성이 있습니다.

반면 else를 사용하지 않은 경우는 컴파일 에러가 나게됩니다.

-> 새로운 sealed 타입이 추가되면 개발자에게 이 케이스에 대한 대응을 해야한다는 책임을 지게할 수 있습니다.

물론 상황에 따라 else가 필요하거나 유용한 경우도 있지만

그런게 아니라면 sealed와 when을 사용할 때는 else 사용을 자제하자는 것이 저의 의견입니다.

## 📌 enum class에 대해 설명해주세요.

enum class는 고정된 상수 집합을 표현합니다. 각 enum 상수는 싱글톤 인스턴스입니다.

프로퍼티와 메서드를 가질 수 있고, abstract 메서드 선언도 가능합니다

## 📌 언제 enum class를 사용하고 언제 sealed class를 사용해야하나요?

enum class는 각 상수가 동일한 타입/형태를 가질 때 적합합니다 (예: 요일, 방향, 상태코드). 

sealed class는 각 서브타입이 서로 다른 데이터를 가질 때 적합합니다 (예: 네트워크 결과, UI 상태). 

enum의 상수는 모두 같은 타입의 인스턴스지만, sealed class의 서브클래스는 각각 다른 프로퍼티를 가질 수 있습니다.
