# 우아한테크세미나 코프링 정리

## 코틀린이란?
* JVM, 안드로이드, 자바스크립트 및 네이티브를 대상으로 하는 `정적 타입 지정 언어`
  * `정적 타입 지정 언어`: 모든 프로그램 구성요소의 타입을 컴파일 시점에 알 수 있고 프로그램 안에서 객체의 필드나 메서드를 사용할 때마다 컴파일러가 타입을 검증해 준다는 뜻
* 젯브레인즈에서 개발한 오픈소스(아파치 라이선스 2.0)
* OO 스타일과 FP 스타일을 모두 사용할 수 있으며 두 요소를 혼합하여 사용할 수 있다.
* 간결하고 실용적이며 안전하고 기존 언어와의 상호 운용성을 중시한다.(+코루틴)

## 코틀린의 역사
* 2011년 7월 코틀린 프로젝트 공개
* 2012년 6월 안드로이드에서 사용 가능
* 2016년 2월 코틀린 1.0 출시 / 스프링 이니셜라이저 지원
* 2017년 1월 스프링 프레임워크 공식 지원
* 2017년 5월 안드로이드 공식 지원
* 2018년 11월 그레이들 코틀린 DSL 1.0 출시
* 2019년 5월 안드로이드 코틀린 퍼스트
* 2019년 9월 스프링 레퍼런스 예제 코드 제공
* 2021년 5월 코틀린 1.5 출시

## 얼마나 사용하고 있나요?
* 지난 12개월 동안 480만 명 이상의 사용자가 사용
* 대다수의 코틀린 개발자는 안드로이드(64%) 및 서버 측 애플리케이션 개발 (52%)에 사용한다
* 대한민국에서 가장 인기가 많은 자바, 자연스럽게 코틀린에 대한 관심도 1위(10%)

## 멀티 플랫폼 언어
* Kotlin은 멀티플랫폼 언어이다.
* JVM/JS/Native 등

## 아이템 1. 코틀린 표준 라이브러리를 익히고 사용하라
* 코틀린 1.3부터 모든 플랫폼에서 사용할 수 있는 `kotlin.random.Random`이 도입되었다.
* 더 이상 `Random`을 사용할지 `ThreadLocalRandom`을 사용할지 고민할 필요가 없다.
* 자바와 관련된 import문을 제거할 수 있다.
* 표준 라이브러리를 사용하면 그 코드를 작성한 전문가의 지식과 여러분보다 앞서 사용한 다른 프로그래머들의 경험을 활용할 수 있다.
* 코틀린은 읽기 전용 컬렉션과 변경 가능한 컬렉션을 구별해 제공한다.
* 인터페이스를 만족하는 실제 컬렉션이 반환된다. 따라서 플랫폼별 컬렉션을 사용할 수 있다.  
![코틀린 컬렉션 프레임워크 다이어그램](images/코틀린-공식문서-컬렉션프레임워크-다이어그램.png)

## 코틀린 맛보기
```kotlin
class Person(val name: String, val age: Int = 1) {
    var nickname: String? = null
}
```
### 자바 변환
```java
public final class Person {
   @NotNull
   private final String name;
   private final int age;
   
   @Nullable
   private String nickname;
   
   public Person(String name) {
      this(name, 1);
   }
   
   public Person(@NotNull String name, int age) {
      this.name = name;
      this.age = age;
   }

   @NotNull
   public final String getName() {
      return this.name;
   }
   
   public final int getAge() {
      return this.age;
   }
   
   @Nullable
   public final String getNickname() {
      return this.nickname;
   }

   public final void setNickname(@Nullable String var1) {
      this.nickname = var1;
   }
}
```

* 코틀린은 기본적으로 널이 될수 없는 타입이고 의도적으로 `?` 키워드를 선언해야 널이 될 수 있는 타입이 된다.
* 자바로 변환했을 때 주요 키워드는 `final`이다.
* 기존 라이브러리를 함께 사용할 때 대부분 `final` 때문에 문제가 발생할 수 있다.

## 아이템 2. 자바로 역컴파일하는 습관을 들여라
* 코틀린 숙련도를 향상시키는 가장 좋은 방법 중 하나는 작성한 코드가 자바로 어떻게 표현되는지 확인 하는 것이다.
* 역컴파일을 통해 예기치 않은 코드 생성을 방지할 수 있다.
* 기존 자바 라이브러리와 프레임워크를 사용하며 문제가 발생할 때 빠르게 확인 할 수 있다.
* IntelliJ IDEA Tools > Kotlin > Show Kotlin Bytecode => Decompile

### 코틀린 컴파일
![코틀린 컴파일](images/코틀린-컴파일.png)
* 코틀린이 먼저 컴파일 된 후 자바가 컴파일 되고, 그 과정에서 애너테이션 프로세싱이 실행된다.
* `애너테이션` -> 가장 자주 쓰는 것은 `롬복` 애너테이션이다.

## 아이템3. 롬복 대신 데이터 클래스를 사용하라
* 데이터를 저장하거나 전달하는 것이 주 목적인 클래스를 만드는 경우가 많다. 이러한 클래스의 일부 표준 및 유틸리티 함수는 데이터에서 기계적으로 파생된다.
* 자바에서는 롬복의 `@Data`를 사용하여 보일러플레이트 코드를 생성한다.
* 애너테이션 프로세서는 코틀린 컴파일 이후에 동작하기 때문에 롬복에서 생성된 자바 코드는 코틀린 코드에서 접근할 수 없다.
* 코틀린 코드보다 자바 코드를 먼저 컴파일하도록 빌드 순서를 조정하면 롬복 문제는 해결할 수 있다.
* 하지만 자바 코드에서 코틀린 코드를 호출할 수 없게 된다.
* 자바에서 코틀린으로 변환 과정에서 추천하는 방법은 Delombok을 사용하여 작은 단위에 데이터 클래스(DTO등) 부터 점진적으로 변경하는 방법을 추천한다

> [코틀린 1.5.20 부터 롬복 컴파일러 프러그인이 실험적으로 추가되었다.](https://kotlinlang.org/docs/lombok.html)

> [Kotlin 도입 과정에서 만난 문제와 해결방법](https://d2.naver.com/helloworld/6685007)

```kotlin
val javajigi = Person(name = "박재성", age = 49)
val jason = javajigi.copy(age = 30)

data class Person(val name: String, val age: Int)
```

```kotlin
data class RecruitmentResponse(
    val id: Long,
    val title: String,
    val term: TermResponse,
    val recruitable: Boolean,
    val hidden: Boolean,
    val startDateTime: LocalDateTime,
    val endDateTime: LocalDateTime,
    val status: RecruitmentStatus
) {
    constructor(recruitment: Recruitment, term: Term) : this(
        recruitment.id,
        recruitment.title,
        TermResponse(term),
        recruitment.recruitable,
        recruitment.hidden,
        recruitment.startDateTime,
        recruitment.endDateTime,
        recruitment.status
    )
}
```

```kotlin
@Embeddable
data class RecruitmentPeriod(
    @Column(nullable = false)
    val startDateTime: LocalDateTime,

    @Column(nullable = false)
    val endDateTime: LocalDateTime
) {
    init {
        require(endDateTime >= startDateTime) { "시작 일시는 종료 일시보다 이후일 수 없습니다." }
    }

    fun contains(value: LocalDateTime): Boolean = (startDateTime..endDateTime).contains(value)
}
```

* 위와 같이 `equals`와 `hashcode` 중요한 객체에서는 데이터클래스를 활용할 수 있다.


## Spring Boot

## 아이템 5. 변경 가능성을 제한하라


## Persistence

* Entity와 MappedSuperclass 를 왜 allOpen 하는지?
* 프록시를 만들 수 없어서 레이지 로딩기능이 안되기 때문에 성능에 문제가 발생한다

## 아이템 6. 엔티티에 데이터 클래스 사용을 피하라
* 롬복의 @Data와 같은 맥락이다. 양방향 연관 관계의 경우 toString(), hashcode()를 호출될 때 무한 순환 참조가 발생한다.

## 아이템 7. 사용자 지정 getter를 사용하라

## 아이템 8. 널이 될 수 있는 타입은 빠르게 제거하라



## 공부방법
* 코틀린 공식문서!!
* 책 - Kotlin in Action 완독 추천
* 아름다움은 기능을 따라간다 - 설리번(건축가)

## 코틀린
* 1.5부터는 IR컴파일러 도입하여 더 성능이 좋다
* 인텔리제이 최신버전 그레이들 최신버전 코틀린 최신버전을 사용하면 3박자가 잘 맞아 재밌게 사용할 수 있다!

## Reference
* [[LIVE 다시보기] 어디 가서 코프링 매우 알은 체하기! : 9월 우아한테크세미나](https://www.youtube.com/watch?v=ewBri47JWII)