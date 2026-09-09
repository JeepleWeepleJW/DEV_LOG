# Kotlin: object 선언 vs companion object

| 구분 | object | companion object |
|---|---|---|
| 설명 | 특정 클래스와 무관하게 독립적으로 존재하는 싱글톤 | 특정 클래스에 종속되어 그 클래스와 함께 존재하는 싱글톤 |
| 접근 방법 | 객체 이름으로 직접 접근<br>(`Manager.method()`) | 클래스 이름으로 접근<br>내부적으로 Companion 인스턴스 경유<br>(`User.create()`) |
| 초기화 시점 | 최초 참조 시점에 해당 클래스가 로드되며 초기화 | companion object가 소속된 class가 로드될 때 초기화 |
| 주요 용도 | 유틸리티 객체<br>전역 상태 관리<br>인터페이스 구현체 | 팩토리 메서드<br>상수 정의<br>클래스 관련 정적 멤버 |
| 장점 | 전역 싱글톤<br>thread-safe<br>lazy 초기화<br>다형성 지원 (구현 시) | 클래스와 논리적 결합 명확<br>소속 클래스의 private 멤버 접근 가능 |
| 단점 | 의존성 주입 어려움<br>테스트 시 mock 대체 어려움<br>참조 보유 시 메모리 누수 위험 | 접근 시 간접 참조 발생 (진짜 static 아님)<br>상속 관계에서 다형적 오버라이드 제약 |

## 참고 코드

```kotlin
// object 선언 - 독립적인 전역 싱글톤
object NetworkManager {
    fun request() { /* ... */ }
}
// 사용: NetworkManager.request()

// companion object - 클래스에 종속된 정적 멤버/팩토리
class User private constructor(val id: String) {
    companion object {
        fun create(id: String): User = User(id)
    }
}
// 사용: User.create("123")
```

## 보충 설명

- **논리적 결합 명확** (가독성/설계 관점): 코드를 읽는 사람이 이 객체가 어떤 클래스와 관련 있는지 이름과 위치만으로 바로 알 수 있음.
- **private 멤버 접근 가능** (컴파일러 차원의 실질적 권한): companion object는 같은 클래스 내부에 선언되어 실제로 그 클래스의 private 생성자·필드에 접근 가능. 독립된 object는 다른 클래스의 private 멤버에 접근 불가.
- **Thread-Safe / lazy 초기화**: JVM의 클래스 초기화(`<clinit>`)가 단 한 번만, 동시 접근 시에도 하나의 스레드만 실행하도록 보장하기 때문에 성립. Kotlin이 별도로 synchronized 처리를 하는 게 아니라 JVM 클래스 로딩 스펙에서 나오는 특성.
