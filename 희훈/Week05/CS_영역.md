# CS 영역
- 프로세스 vs 스레드, 안드로이드 메인 스레드/UI 스레드 동작
- 가비지 컬렉션 (ART의 동작, Generational GC)
- HTTPS 동작 원리, JWT, OAuth 2.0 흐름
- HashMap (equals / hashCode 계약)

# 프로세스 vs 스레드, 안드로이드 메인 스레드/UI 스레드 동작

## 1. 프로세스와 스레드의 차이

### 프로세스 (Process)
- 정의 - 실행 중인 프로그램의 인스턴스
- 메모리 - 독립적 (Code, Data, Heap, Stack)
- 통신 - IPC(Inter-Process Communication) 필요 (파이프, 소켓, 공유메모리, Binder)
- 생성 비용 - 비쌈
- 안정성 - 한 프로세스 죽어도 다른 프로세스 영향 없음
- Context Switch - 비용 큼 (메모리 매핑 교체)

### 스레드 (Thread)
- 정의 - 프로세스 안의 실행 단위
- 메모리 - Code/Data/Heap 공유, Stack만 별도
- 통신 - 같은 메모리 직접 접근
- 생성 비용 - 프로세스에 비해 비교적 가벼움
- 안정성 - 한 스레드 크래시 -> 프로세스 전체 영향
- Context Switch - 비용 작음

## 2. 안드로이드의 프로세스 모델
- 각 앱은 기본적으로 하나의 리눅스 프로세스 실행됨

## 3. 메인 스레드 (UI 스레드)
앱 프로세스가 시작되면 시스템이 메인 스레드를 하나 만듬

- 모든 UI 작업(View 그리기, 측정, 이벤트 처리)은 이 스레드에서만 가능
- 16ms마다 한 프레임을 그려야 함 (60Hz 기준) -> 그래서 메인 스레드는 차단되면 안 됨
- 메인 스레드는 내부적으로 Looper + MessageQueue + Handler 로 동작

## 4. Looper / MessageQueue / Handler
- Looper - 무한 루프를 돌며 MessageQueue에서 메시지를 꺼내 dispatch
- MessageQueue - 메시지의 시간순(예약 시간) 우선순위 큐
- Handler - 메시지를 큐에 넣고 처리하는 인터페이스

다른 스레드에서 UI 작업이 필요할 때 메인 스레드의 Handler에 메시지를 넣어 처리하게 함

코루틴의 Dispatchers.Main도 결국 이 메커니즘을 활용

## 5. ANR과의 연결
- 메인 스레드가 5초간 입력 처리를 못 하면 ANR
- 모든 무거운 작업은 다른 스레드로 옮겨야 함

# 가비지 컬렉션 (ART의 동작, Generational GC)

## 1. ART와 Dalvik
- Dalvik (~Android 4.4) - 매 실행 시 JIT(Just-In-Time) 컴파일, GC도 단일 스레드
- ART (Android 5.0~) - 설치 시 AOT(Ahead-Of-Time) 컴파일, GC도 개선됨, Android 7부터는 JIT+AOT 하이브리드

## 2. Generational GC의 원리
- 대부분의 객체는 수명이 짧음 - weak generational hypothesis
- 이 가설을 활용해 힙을 세대로 나눔
- Young Generation - 새로 만든 객체. 자주, 빠르게 수집
- Old Generation - 여러 GC를 살아남은 객체. 드물게, 깊게 수집

### 장점
- Young만 자주 돌리면 대부분 객체가 청소됨 - 빠르고 짧은 GC
- Old는 길게 살 객체들이라 자주 검사할 필요 없음

## 3. Concurrent Copying
- ART 8.0+ 의 기본

### 핵심
1. Concurrent - 앱 스레드를 정지시키지 않고 GC 실행
2. Copying - 살아있는 객체를 다른 영역으로 복사하면서 단편화 제거

### 장점
- pause time이 짧아 끊김 감소
- 단편화가 없어 할당이 빠름

### 단점
- 복사를 위해 메모리 오버헤드

## 4. GC 관련 메모리 누수 영향
GC가 회수하지 못하는 객체가 늘어나면
1. 사용 가능 메모리 감소
2. GC 호출 빈도 증가
3. GC가 길어지면 16ms 프레임 초과 - 화면 끊김
4. 최종적으로 OutOfMemoryError 또는 시스템에 의해 앱 종료

## 5. GC 최적화 코드 작성
- 빈번한 객체 생성 피하기
- 큰 객체는 풀로 재사용
- 약한 참조(WeakReference) 적절히 활용

# HTTPS 동작 원리, JWT, OAuth 2.0 흐름

## 1. HTTPS
- HTTPS는 HTTP를 TLS(과거 SSL) 위에 얹은 프로토콜

### TLS가 하는 일
1. 인증(Authentication) - 서버가 진짜 그 도메인인가
2. 기밀성(Confidentiality) - 데이터가 도청 불가능
3. 무결성(Integrity) - 데이터가 변조되지 않았는가

## 2. 대칭키 vs 비대칭키
- 대칭키 - 같은 키로 암호화/복호화, 빠르지만 키 전달이 문제
- 비대칭키(공개키) - 공개키로 암호화, 개인키로만 복호화 느리지만 키 전달 문제 해결

TLS는 두 가지를 결합 -> 핸드셰이크에서 비대칭키로 안전하게 대칭키를 합의, 이후 본 통신은 빠른 대칭키로 암호화

## 3. TLS 1.3의 개선
- 핸드셰이크가 1-RTT 이전 RTT 절반
- 0-RTT 재개 - 이전 연결 정보로 즉시 데이터 전송
- 약한 cipher suite 제거, 무결성 보장 알고리즘 일원화

## 4. 인증서 검증

### 서버가 보낸 인증서가 신뢰할 만한지
1. 인증서가 신뢰된 CA의 개인키로 서명되었는가 (CA의 공개키로 검증)
2. 인증서가 만료되지 않았는가
3. 인증서의 도메인이 접속한 도메인과 일치하는가
4. 인증서가 폐기되지 않았는가 (OCSP, CRL)

## 5. 안드로이드에서의 TLS
- OkHttpClient는 기본적으로 시스템의 trust store를 사용
- 자체 서명 인증서나 사설 CA를 쓰려면 TrustManager 커스텀 또는 Network Security Configuration으로 처리

Certificate Pinning은 유용하지만, 인증서 갱신 시 앱 업데이트가 필요함

## 6. JWT (JSON Web Token) 구조
- Header.Payload.Signature 형식
- 각 부분은 Base64URL 인코딩

## 7. JWT 핵심
- JWT는 암호화되지 않음 - payload는 누구나 디코딩해서 볼 수 있음, 비밀 정보 X
- 서명은 위변조 검증용 - 서버가 secret(또는 공개키)로 서명만 확인하면 신뢰 가능
- Stateless - 서버가 세션 저장소 없이도 인증 정보를 검증할 수 있음
- 만료 처리는 클라이언트가 갱신, 서버는 거부 정도만 가능, 즉시 무효화는 어려움

### Access Token + Refresh Token
- Access Token - 짧은 수명 (5분 ~ 1시간), API 호출에 사용
- Refresh Token - 긴 수명 (며칠 ~ 수십일), 새 Access Token 발급용

### 안드로이드 저장 위치
- EncryptedSharedPreference 권장 - 키스토어로 보호된 저장소
- DataStore + EncryptedSharedPreferences 결합도 가능

## 8. OAuth 2.0
- OAuth는 인가(Authorization) 프로토콜
- 사용자의 자격증명을 직접 알지 못해도, 사용자가 허락한 범위(scope) 내에서 자원에 접근할 수 있게 해줌

### OpenID Connect (OIDC)
- OAuth 2.0 위에 인증(Authentication)을 더한 것
- id_token(JWT 형식)을 추가로 발급해 사용자 신원을 증명

# HashMap (equals / hashCode 계약)

## 1. HashMap의 동작 원리
- HashMap은 키의 해시값으로 버킷을 결정
- 같은 버킷 안에서 equals로 정확한 키를 찾음
- 내부적으로 자바 8+에서는 한 버킷의 충돌이 일정 임계(8)를 넘으면 LinkedList → 레드-블랙 트리로 바뀜
  - O(n) → O(log n) 보장

## 2. equals와 hashCode 계약
- Object#equals와 Object#hashCode는 하기 규칙 만족해야함

### equals 계약
1. 반사성(reflexive) - x.equals(x) == true
2. 대칭성(symmetric) - x.equals(y) == y.equals(x)
3. 추이성(transitive) - x.equals(y) && y.equals(z) -> x.equals(z)
4. 일관성(consistent) - 호출할 때마다 같은 결과
5. x.equals(null) == false

### hashCode 계약
1. 두 객체가 equals이면 hashCode도 같아야 함
2. equals가 다른 두 객체의 hashCode는 같아도 됨
3. 같은 객체에 여러 번 호출하면 같은 값

## 3. 계약을 어기면 어떻게 되는가
- equals로는 같은 키지만 hashCode가 달라서 버킷을 못 찾음
- HashMap의 키는 불변(immutable)이어야 하며 String, 숫자, data class(필드 변경 안 하는 경우)가 안전

## 4. 계약 관련 함정

### equals만 같으면 되지, hashCode는 왜
- HashMap/HashSet은 hashCode로 버킷을 먼저 찾고 equals로 비교
- hashCode가 다르면 같은 객체로 취급되지 않으니 컬렉션 동작이 깨짐

### hashCode를 항상 0으로 반환
- 모든 객체가 같은 버킷에 들어가서 충돌 폭주
- O(1)이 아니라 O(n)이 됨

### set에 넣고 나서 객체 필드를 바꾸면
- hashCode가 바뀌어 set이 그 객체를 영영 못 찾게 됨
- 컬렉션 키/원소는 불변으로 다뤄야 함

# 참고 자료
- [Android Source - ART 가비지 컬렉션 디버그](https://source.android.com/devices/tech/dalvik/gc-debug)
- [Google for Developers - OAuth 2.0](https://developers.google.com/identity/protocols/oauth2)
