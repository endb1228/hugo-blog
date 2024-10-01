+++
title = '[Spring] Fixture Monkey'
date = 2024-09-05T17:11:44+09:00
draft = false
+++

### 테스트 케이스 작성이 어려운 이유
완전하게 TDD로 개발하고 있지는 않지만,
필요하다고 생각되는 테스트 케이스는 항상 개발과 병행해서 작성하는 게 습관화된 것 같다.
그럼에도 익숙해지지 않는 것은 test fixture를 생성하는 것.

test fixture를 생성할 때마다 고민되는 부분이 크게 두 가지가 있다.
> - 필드의 setter가 private한 경우
> - 연관관계가 복잡하게 얽혀있는 경우

테스트 케이스를 작성한다는 것은 테스트가 필요한 클래스에 대해 임의의 인스턴스(test fixture)를 생성해야 한다.
하지만 인스턴스의 setter가 private한 경우에는 데이터를 임의로 조작하기 어려워진다.
(setter만을 언급하긴 했지만 생성자나 빌더가 닫혀있는 경우 등, 원하는 test fixture의 데이터를 조작하기 어렵게 만드는 것들.)

그래서 setter든 생성자든 접근 제어자에 관계없이 test fixture를 생성하기 위해서 지금까지는 reflection을 사용했다.
하지만  도메인이나 dto가 추가될 때마다 test fixture를 새로 생성해야 하고,
연관관계가 생기면 연관된 모든 클래스의 test fixture를 다시 수정해야 하는 번거로움이 생기게 된다.
그리고 연관관계가 있으면 연관관계에 있는 클래스에 대한 test fixture도 또 생성해줘야 한다.
여러 test fixture들에 대해 reflection을 모두 적용하기 위해 별도의 유틸리티 함수들을 생성하는 등,
test fixture 생성에 드는 시간을 줄이기 위해 여러 방법들을 사용해봤지만 설계도 점점 복잡해지고 오래 걸리는 것은 마찬가지였다.
코딩의 목적이 테스트는 아니기 때문에, 테스트에 들어가는 시간보다 메인 코드에 더 많은 시간을 할애해야 한다고 생각하는 입장에서는
항상 프로젝트를 진행할 때마다 걸림돌이 되는 것 같았다.

### test fixture 손쉽게 생성해보자!

원래 test fixture라는 용어는 모르고 있었다. 항상 '가짜 데이터'라는 생각에 'mock data'라는 용어를 쓰고,
클래스 명에도 앞에 mock이라는 단어를 붙여서 사용했다.

프로젝트를 진행하면서 테스트에 필요한 임의의 데이터는 test fixture라고 부르는 것을 알게 됐고,
위의 고민처럼 test fixture를 쉽게 생성할 수 있는 방법이 없을까 고민하던 중 Fixture Monkey라는 라이브러리를 알게 됐다.
(아마 test fixture라는 용어를 몰랐으면 찾지 못 했을 수도?)

### Fixture Monkey?

Fixture Monkey는 네이버 오픈소스에서 개발한 Java/Kotlin 라이브러리다.  

**"Write once, Test anywhere"** 라는 문장에서 알 수 있듯,
한 번 작성하는 것만으로 모든 테스트에서 쉽게 사용할 수 있도록 하는 라이브러리다.
따로 명시해주지 않으면 모든 필드를 타입에 맞춰서 임의의 데이터를 자동으로 생성해준다.
### Fixture Monkey의 test fixture 생성 전략

Fixture Monkey는 여러 가지 전략으로 test fixture를 생성할 수 있다.
Fixture Monkey는 `Introspector`라는 클래스로 전략을 설정한다.

그 중에서 내가 test fixture를 생성할 때 가장 문제가 됐었던 게 접근 제어자였기 때문에,
그에 관계없이 test fixture를 생성하기 위해서 `FieldReflectionArbitraryIntrospector`를 사용했다.
클래스 이름에서 알 수 있듯이, reflection을 사용해서 test fixture를 생성한다.
그리고 이러한 설정이 대부분의 test에서 공통적으로 사용될 거라 생각하고 configuration 및 bean으로 설정해주었다.

그렇게 개발을 시작하고 보니 내 임의로 test fixture를 생성하기 위해 사용했던 클래스들은 모두 제거할 수 있었고,
새로 생성한 클래스라고는 configuration 클래스 하나였다. 그리고 코드도 가독성이 훨씬 좋아진 게 느껴졌다.

개발 도중에 `LocalDateTime` 타입의 필드에 `@JsonFormat`으로 형식을 설정해 주었는데,
FixtureMonkey가 이를 무시하는 문제가 있었다.
그래서 jackson 라이브러리를 인식하기 위한 `JacksonObjectArbitraryIntrospector`,
그리고 둘 이상의 `Introspector`를 사용하기 위한 `FailoverArbitraryIntrospector`를 설정에 추가해주었다.
그래서 최종적으로 Fixture Monkey에 대한 기본 설정은 다음과 같이 구성했다.

```java
@TestConfiguration
public class FixtureMonkeyConfig {

    @Bean
    public FixtureMonkey defaultFixtureMonkey() {
        return FixtureMonkey.builder()
                .objectIntrospector(
                        new FailoverIntrospector(Arrays.asList(
                                JacksonObjectArbitraryIntrospector.INSTANCE,
                                FieldReflectionArbitraryIntrospector.INSTANCE
                        ))
                )
                .build();
    }
}
```
`FailOverArbitraryIntrospector`에서 배열 내부의 설정들은 앞에서부터 적용된다고 보면 된다.
먼저 앞에 있는 설정이 높은 우선순위로 적용되고, test fixture 생성에 문제가 발생하면 다음 설정을 적용한다.
(내 경우에는 jackson 라이브러리가 적용된 필드를 정상적으로 생성하기 위해 jackson 관련 설정을 먼저 추가해주었고,
만약 접근 제어자로 인한 필드 생성에 문제가 발생하면 reflection으로 생성하도록 설정해주었다.)

### Fixture Monkey 적용 후기

Fixture Monkey의 장점은 **'쉽고 간단하다'** 는 말로 요약할 수 있을 것 같다.
위에서 언급한 기본 설정 외에도 엣지 케이스를 생성하는 방법도 매우 쉽고 간편하다.

사용한 경험이 아직 많진 않지만, 가장 아쉬웠던 것은 `String`타입의 임의의 데이터에 중국어가 들어가는 것?
언어 등 추가해줄 수 있는 기본 설정들이 많을 것 같은데,
아직 설정에 대한 기능 지원이 많이 부족해 각각의 데이터의 format을 확실하게 정해줘야 테스트가 의도대로 작동하는 것 같다.
