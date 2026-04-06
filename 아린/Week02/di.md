# 안드로이드 DI & Hilt

## Q. 의존성 주입(Dependency Injection) 이란?

- 객체가 필요한 부품을 직접 생성하지 않고 **외부에서 주입**받는 디자인 패턴

### DI가 왜 필요할까?

- **결합도 감소**
    - 클래스 간 의존성을 낮춤으로써 코드 변경이 쉬워진다
- **재사용성 향상**
    - 동일한 부품을 여러 곳에서 쉽게 갈아 끼울 수 있다
- **테스트 용이성**
    - 실제 객체 대신 Mock 객체를 주입하여 유닛 테스트에 용이하다

---

## Q. Hilt란?

- 구글이 권장하는 안드로이드 전용 DI 라이브러리
- 가장 강력한 DI 도구인 Dagger를 기반으로 만들어졌으며,
  Dagger의 복잡한 초기 설정과 보일러플레이트 코드를 안드로이드 환경에 맞게 자동화하여 사용성을 극대화한 프레임워크

## Q. 왜 Hilt여야 하는가

- **안드로이드 생명주기(Lifecycle) 자동 관리**
    - 순수 Dagger → 개발자가 Activity나 Fragment의 생명주기에 맞춰 DI 컨테이너를 생성하고 해제하는 코드를 직접 짜야 함
    - Hilt → 안드로이드의 표준 컴포넌트(Application, Activity, Fragment, ViewModel 등)에 맞춰 미리 정의된 컨테이너들이 있음
        - 어노테이션만 붙이면 시스템이 알아서 해당 화면이 켜질 때 객체를 주입하고, 파괴될 때 메모리에서 정리한다
          ex ) `@AndroidEntryPoint`
- **극단적인 보일러플레이트 코드 감소**
    - 순수 Dagger → Component 인터페이스를 만들고, 모듈을 연결하고, 주입할 타겟을 일일이 명시해야 함
    - Hilt → KAPT/KSP를 통해 컴파일 타임에 이 모든 Dagger 세팅 코드를 백그라운드에서 **자동 생성**
- HiltViewModel
    - `@HiltViewModel` 어노테이션 하나만으로 팩토리 클래스 생성 없이 뷰모델 주입 처리
- **컴파일 타임 DI**
    - Reflection을 사용하여 성능을 깎아먹는 일부 DI 도구들과 달리,
      Hilt는 앱을 빌드하는 컴파일 타임에서 주입에 필요한 모든 코드를 미리 완성한다
      → 앱 실행 속도에 악영향을 주지 않고 타입 안전성을 보장한다

---

## Q. 핵심 어노테이션

### `@HiltAndroidApp`

앱 전체에서 Hilt를 사용하겠다고 선언하는 **메인 스위치**

- 어디?
    - 안드로이드의 `Application` 클래스 위
- 역할?
    - Hilt가 앱 전체의 의존성을 관리할 최상위 컨테이너를 자동으로 만든다
    - 이 어노테이션이 없으면 Hilt의 모든 기능이 동작하지 않는다

### `@AndroidEntryPoint`

Hilt가 만든 객체들을 여기서 받아 쓰겠다 라고 선언하는 컴포넌트 준비 구간

- 어디?
    - `Activity`, `Fragment`, `Service`, `BroadcastReceiver` 같은 안드로이드 생명주기를 가진 컴포넌트 위
- 역할?
    - 위 안드로이드 시스템이 생성하는 컴포넌트들에게 Hilt가 접근해서 **객체를 주입할 수 있도록 연결 통로**를 연다

### `@Inject`

1. 생성자 주입

- 개발자가 직접 만든 클래스의 생성자에 붙여 Hilt에게 이 객체를 어떻게 생성하는지 알려줍니다.
1. 필드 주입
- `@AndroidEntryPoint`가 붙은 액티비티나 프래그먼트 안에서 필요한 객체를 요청

### `@HiltViewModel`

`ViewModel`을 Hilt로 편하게 다루기 위해 만들어진 어노테이션

- 어디?
    - `ViewModel` 클래스 위에 붙입니다.
- 역할?
    - 기존에 뷰모델 생성자에 Repository 등의 인자를 넘겨주려면 팩토리를 생성해 주는 복잡한 코드를 짜야 하지만,
      이 어노테이션과 `@Inject constructor`를 함께 쓰면 Hilt가 뷰모델 팩토리 생성을 알아서 뒤에서 다 처리한다

> 아래 4개의 어노테이션은 Retrofit, Room Database 등과 같이 내가 직접 생성자에 `@Inject`를 달 수 없는 외부 라이브러리 클래스, 인터페이스들의 생성 방법을 Hilt에게 알려준다
>

### `@Module`

객체 생성 설명서를 작성하겠다는 선언

### **`@InstallIn`**

어떤 컴포넌트에 넣을지 결정
ex) SingletonComponent

### **`@Provides`**

Module 어노테이션이 붙은 클래스 내부의 함수에 사용한다.
이 모듈안에서 구체적으로 객체를 리턴하는 함수 위에 붙여 생성 방법을 작성한다

### **`@Binds`**

인터페이스와 실제 구현체를 연결할 때 사용한다

### 기초 설정

| **어노테이션** | **설명** | **위치** |
| --- | --- | --- |
| **`@HiltAndroidApp`** | Hilt의 시작점으로 앱 전체 컨테이너(Singleton) 생성 | `Application` 클래스 |
| **`@AndroidEntryPoint`** | 의존성을 주입받을 안드로이드 컴포넌트 지정 | `Activity`, `Fragment` 등 |

### 주입 및 생성

| **어노테이션** | **설명** | **위치** |
| --- | --- | --- |
| **`@Inject`** | **1. 주입 요청:** "이 객체 주입해" (필드 주입)
**2. 생성법 등록:** "나를 사용할 땐 이렇게 만들어" (생성자 주입) | 변수 위 / 생성자 앞 |
| **`@HiltViewModel`** | ViewModel을 Hilt가 관리하도록 지정. | `ViewModel` 클래스 |

### 외부 라이브러리 & 인터페이스 (Module)

| **어노테이션** | **설명** | **위치** |
| --- | --- | --- |
| **`@Module`** | 외부 라이브러리나 인터페이스 생성 설명서임을 명시. | `object` 또는 `class` |
| **`@InstallIn`** | 모듈이 유지될 범위(Component) 결정 (예: `SingletonComponent`). | `@Module`과 함께 사용 |
| **`@Provides`** | Retrofit 등 생성자 주입이 불가능한 객체의 생성법 작성. | 모듈 내 함수 |
| **`@Binds`** | 인터페이스와 실제 구현체를 연결할 때 사용. (추상 함수 사용) | 모듈 내 함수 |

---

## Q. Hilt 주입 흐름

1. `Application`에 `@HiltAndroidApp` 달기
2. **등록**
    - 내 클래스 → `@Inject constructor`
    - 외부 라이브러리 → `@Module` + `@Provides`
3. 화면(`Activity`, `Fragment`)에 `@AndroidEntryPoint` 달기
4. **객체 조립 및 사용:** 액티비티에서는 `@Inject`로, 뷰모델은 `@HiltViewModel`을 통해 알아서 주입받아 사용하기

## **Q. Hilt의 Component 계층 구조**

상위 컴포넌트의 객체는 하위 컴포넌트에서 자유롭게 가져다 쓸 수 있는 하향식 구조

`SingletonComponent` → `ActivityRetainedComponent`-> `ViewModelComponent` → `ActivityComponent` → `FragmentComponent`

## **Q. 왜 Dagger2 보다 굳이 Hilt를 권장하는가**

Dagger2는 러닝 커브가 너무 높고 설정 코드가 비대해져 유지보수가 어렵다…

반면에 Hilt는 Dagger의 뛰어난 컴파일 타임 성능과 안정성을 그대로 가져오되
안드로이드 생명주기 관리를 자동화해주어 개발 생산성을 높인다