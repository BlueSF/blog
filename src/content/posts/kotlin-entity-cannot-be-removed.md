---
author: 위영종
pubDatetime: 2026-08-16T22:00:00+09:00
title: "Kotlin에서 @Entity는 지울 수 없다"
description: "JPA 매핑을 orm.xml로 옮겨도 @Entity는 클래스에 남는다. 컴파일러 플러그인이 이유이고, Kotlin 2.3.20부터 동작이 바뀌었다."
featured: false
draft: false
tags: ["kotlin", "jpa", "hibernate", "hexagonal-architecture", "orm-xml"]
---

JPA 매핑을 `orm.xml`로 전부 옮겨도 Kotlin에서는 `@Entity`를 지울 수 없다. 컴파일러 플러그인의 트리거이기 때문이고, XML은 컴파일러에게 아무 말도 걸 수 없기 때문이다.

이 글은 그 구조를 확인한 기록이다. 왜 매핑을 XML로 분리하려 했는지는 [도메인 클래스에 무엇을 허용할 것인가](/posts/what-to-allow-in-domain-class/)에서 다뤘다.

JPA는 XML 매핑을 정식으로 지원한다. `META-INF/orm.xml`에 매핑을 선언하면 어노테이션 없이도 엔티티를 매핑할 수 있고, 같은 속성이 양쪽에 있으면 **XML이 이긴다**. `metadata-complete="true"`를 주면 아예 어노테이션을 전부 무시시킬 수도 있다.

그래서 이렇게 시작한다.

```kotlin
// Before
@Entity
@Table(name = "orders", indexes = [Index(name = "idx_user", columnList = "user_id")])
class Order(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long? = null,
    @Column(name = "user_id", nullable = false)
    val userId: Long,
)
```

```kotlin
// After — 상세 매핑은 XML로
@Entity
class Order(
    @Id val id: Long? = null,
    val userId: Long,
)
```

여기까지는 잘 된다. 그런데 **끝까지 밀어붙이면 어느 지점에서 멈춘다.** `@Entity`와 `@Id`마저 지우려는 순간부터 설명하기 어려운 실패들이 나온다.

## 목차

---

## 왜 지울 수 없나 — 컴파일러 플러그인

Kotlin으로 JPA를 쓰면 보통 이 플러그인을 넣는다.

```kotlin
plugins {
    kotlin("plugin.jpa")     // no-arg 합성 + final 해제 (2.3.20+, 뒤에서 확인한다)
    kotlin("plugin.spring")  // Spring 빈의 final 해제. 엔티티와는 무관하다
}
```

`kotlin-jpa`는 no-arg 플러그인을 JPA용으로 특수화한 것이다. 어떤 어노테이션을 트리거로 삼는지는 플러그인 jar 안에 그대로 들어 있다.

```
$ unzip -p kotlin-noarg-compiler-plugin-embeddable-<version>.jar \
      org/jetbrains/kotlin/noarg/NoArgPluginNames.class | strings
jpa
javax.persistence.Entity      javax.persistence.Embeddable      javax.persistence.MappedSuperclass
jakarta.persistence.Entity    jakarta.persistence.Embeddable    jakarta.persistence.MappedSuperclass
```

이 세 어노테이션이 붙은 클래스에 파라미터 없는 생성자를 합성해 준다. Kotlin은 주 생성자에 파라미터가 있으면 no-arg 생성자가 존재하지 않고, Hibernate는 리플렉션으로 no-arg 생성자를 호출해 인스턴스를 만들기 때문에 이게 없으면 엔티티를 만들 수 없다.

여기서 핵심은 이것이다.

> **컴파일러 플러그인은 `orm.xml`을 읽지 않는다. 읽을 수도 없다.**
> XML은 런타임에 Hibernate가 파싱하는 리소스이고, 플러그인은 컴파일 시점에 소스 트리만 본다.
> **따라서 클래스에는 어떤 형태로든 어노테이션이 남는다.**

도입부에서 언급한 `metadata-complete="true"`도 여기서는 소용이 없다. 그건 **런타임에 Hibernate가 어노테이션을 읽지 않게 하는** 스위치다. 컴파일 시점에 플러그인이 어노테이션을 보는 것과는 무관하다. 매핑 메타데이터로서의 역할은 무시시킬 수 있어도, 컴파일러 스위치로서의 역할은 남는다.

그리고 문장을 "어떤 형태로든"이라고 쓰는 데는 이유가 있다. 정확히 말하면 필요한 것은 `@Entity` **자체가 아니다.** noarg와 allopen 플러그인은 프리셋 대신 임의의 어노테이션을 지정할 수 있다.

```kotlin
noArg { annotation("com.example.NoArgConstructor") }   // 내가 만든 어노테이션도 트리거가 된다
```

그러니 이론상 `@Entity`를 떼고 자체 마커 어노테이션을 붙이는 것도 가능하다. 하지만 그건 **어노테이션을 다른 어노테이션으로 바꾼 것**이지, 클래스에서 어노테이션을 없앤 게 아니다. 게다가 그 대가로 IDE 지원까지 잃는다. 실익이 없다.

즉 문제의 정체는 "`@Entity`가 특별해서"가 아니라 **"XML은 컴파일러에게 아무 말도 걸 수 없어서"** 다.

### 버전을 바꿔 보면 명확해진다

`@Entity` 하나만 다른 두 클래스를 준비했다.

```kotlin
@Entity
class Order(
    @Id val id: Long? = null,
    val userId: Long,
)

class OrderSummary(
    val id: Long? = null,
    val userId: Long,
)
```

이걸 Kotlin 2.2.20과 2.3.21로 각각 컴파일하고 `javap`로 열어 봤다.

```
=== Kotlin 2.2.20 ===
[Order       ]  public final class lab.jpa.Order          ← ACC_FINAL 있음
[OrderSummary]  public final class lab.jpa.OrderSummary   ← ACC_FINAL 있음
[Order       ]  no-arg 생성자: 있음
[OrderSummary]  no-arg 생성자: 없음

=== Kotlin 2.3.21 ===
[Order       ]  public class lab.jpa.Order                ← ACC_FINAL 없음
[OrderSummary]  public final class lab.jpa.OrderSummary   ← ACC_FINAL 있음
[Order       ]  no-arg 생성자: 있음
[OrderSummary]  no-arg 생성자: 없음
```

읽는 방법은 두 축이다.

**가로 — `@Entity`가 만드는 차이.** no-arg 생성자는 두 버전 모두에서 갈린다. 이건 예전부터 `kotlin-jpa`가 하던 일이다.

**세로 — 버전이 만드는 차이.** 2.3.21에서 `Order`의 `ACC_FINAL`만 사라졌다. 대조군인 `OrderSummary`는 두 버전 모두 그대로다. 즉 컴파일러 전반의 변화가 아니라 **`@Entity`가 트리거하는 동작이 하나 늘어난 것**이다.

`allOpen { ... }` 설정은 어디에도 없다. `kotlin("plugin.jpa")` 한 줄이 전부다.

### 경계는 정확히 어디인가

두 버전만 비교하면 "2.3 어딘가에서 바뀌었다"까지밖에 말할 수 없다. 사이 버전을 전부 돌려 봤다.

| Kotlin | no-arg 생성자 합성 | `ACC_FINAL` 해제 | 비고                        |
| ------ | ------------------ | ---------------- | --------------------------- |
| 2.2.20 | O                  | X                | `allOpen {}` 직접 선언 필요 |
| 2.3.0  | O                  | X                | 〃                          |
| 2.3.10 | O                  | X                | 〃                          |
| 2.3.20 | O                  | **O**            | **경계.** jpa 프리셋 자동 등록 |
| 2.3.21 | O                  | O                |                             |

**"2.3부터"가 아니라 "2.3.20부터"다.** 2.3.0이나 2.3.10을 쓰고 있다면 메이저·마이너가 올라갔어도 여전히 `allOpen {}`이 필요하다. 마이너 단위로 어림잡으면 정확히 틀리는 구간이다.

### 왜 final이 문제인가

Kotlin 클래스는 기본이 `final`이다. Hibernate는 지연 로딩을 위해 엔티티의 **서브클래스 프록시**를 만들어야 하므로, final 엔티티는 프록시를 만들 수 없다. `@ManyToOne(fetch = LAZY)`를 걸어도 의도대로 동작하지 않는다.

그래서 2.3.20 이전에는 이 설정을 직접 넣어야 했다.

```kotlin
allOpen { annotation("jakarta.persistence.Entity") }
```

이 변화는 [KT-28594](https://youtrack.jetbrains.com/issue/KT-28594)와 KT-79389가 2.3.20에 반영되면서 생겼다. KT-28594는 6년 넘게 열려 있던 이슈다. JetBrains가 2026년 1월에 낸 글에도 "`plugin.jpa`는 필요한 설정을 다 해주는 것처럼 보이지만 no-arg만 설정하고 all-open은 설정하지 않는다"고 적혀 있다. 그때는 맞는 설명이었다.

**그러니 이 글을 읽고 "`plugin.jpa`가 알아서 열어주는구나" 하고 넘어가면 안 된다.** 2.3.20 미만에서는 지연 로딩이 조용히 깨진 채로 간다. 자기 프로젝트의 Kotlin 버전부터 확인할 것.

> 위 출력은 [blog-lab/001-jpa-entity-final](https://github.com/BlueSF/blog-lab/tree/main/001-jpa-entity-final)에서 `./verify.sh --compare`로 재현할 수 있다. 버전별 표는 `./verify.sh --kotlin 2.3.10`처럼 버전을 바꿔 가며 얻은 것이다. JDK 17 이상만 있으면 된다.

### 여담 — 이 문단이 가장 오래 걸렸다

처음엔 all-open의 spring 프리셋 목록만 보고 막혔다. 거기에 `jakarta.persistence.Entity`가 없었기 때문이다.

```
$ unzip -p kotlin-allopen-compiler-plugin-embeddable-<version>.jar \
      org/jetbrains/kotlin/allopen/AllOpenPluginNames.class | strings
spring
org.springframework.stereotype.Component
org.springframework.transaction.annotation.Transactional
...
```

"프리셋에 없는데 왜 열려 있지?"에서 한참을 헤맸다. 같은 파일 뒤쪽에 **jpa 프리셋이 따로 있었는데** grep 범위를 좁게 잡은 탓이다. 결국 Gradle 플러그인 진입점을 디컴파일하고서야 풀렸다.

```java
// KotlinJpaSubplugin.apply()
plugins.apply(NoArgGradleSubplugin.class)      // 1. no-arg 플러그인 적용
plugins.apply(AllOpenGradleSubplugin.class)    // 2. all-open 플러그인도 적용
noArgExtension.myPresets.add("jpa")            // 3. no-arg에 jpa 프리셋 등록
allOpenExtension.preset("jpa")                 // 4. all-open에도 jpa 프리셋 등록  ← 이것
```

참고로 우리 프로젝트의 버전 카탈로그에는 이렇게 적혀 있었다.

```toml
kotlin-jpa = { ... }      # JPA 엔티티 지원 (noarg 자동화)
kotlin-allopen = { ... }  # 클래스 open 처리 (Spring 프록시용)
```

내가 쓴 주석이다. 그리고 2.3.20 전까지는 정확한 설명이었다. **문서가 틀렸던 게 아니라, 산출물이 문서보다 먼저 바뀐 것이다.**

---

## Java와 Kotlin은 결정적으로 다르다

이 문제는 **Kotlin에만 있다.** 이게 생각보다 큰 차이를 만든다.

| 항목                         | Java                                                                                                            | Kotlin                                                                                                                                                     |
| ---------------------------- | --------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **no-arg 생성자**            | 명시적 생성자가 없으면 컴파일러가 자동 생성. 있어도 `protected Order() {}` 한 줄이면 끝 — **어노테이션과 무관** | 주 생성자에 파라미터가 있으면 존재하지 않음. 플러그인이 **어노테이션을 보고** 합성                                                                         |
| **final / 지연 로딩 프록시** | 클래스·메서드 기본 non-final → 프록시 정상                                                                      | 기본 final → all-open 플러그인이 어노테이션 기준으로 해제. **2.3.20부터 `kotlin-jpa`가 jpa 프리셋을 자동 등록하며, 그 이전 버전에서는 수동 등록이 필요**   |
| **접근 타입(access type)**   | 필드에 붙이면 FIELD, getter에 붙이면 PROPERTY. 위치가 곧 의미                                                   | 프로퍼티가 항상 getter를 동반해 모호. `@Access(FIELD)`나 XML `<access>`로 명시하는 편이 안전                                                               |
| **어노테이션 적용 대상**     | 붙인 위치가 곧 대상                                                                                             | 생성자 파라미터의 어노테이션은 param→property→field 순으로 해석. JPA 어노테이션은 PARAMETER 타깃이 없어 field로 떨어짐 — 동작은 하지만 규칙이 눈에 안 보임 |

Java 진영에서는 no-arg 생성자를 이렇게 해결한다.

```java
@NoArgsConstructor(access = AccessLevel.PROTECTED)   // Lombok
@Entity
public class Order { ... }
```

생성자 확보가 **JPA 어노테이션과 완전히 분리**되어 있다. 그래서 Java 엔티티는 `@Entity`까지 지우고 XML만으로 매핑하는 게 가능하다.

이건 추론으로 남기지 않고 돌려 봤다. `jakarta.persistence` import도 어노테이션도 없는 Java 클래스에 매핑을 전부 `orm.xml`로 주고, 클래스패스 스캔을 끈 채(`<exclude-unlisted-classes>true</exclude-unlisted-classes>`) Hibernate를 띄웠다. 스캔이 켜져 있으면 "어노테이션 덕에 잡힌 것 아니냐"를 배제할 수 없기 때문이다.

```
[1] Order 클래스의 어노테이션 개수: 0
[2] 메타모델에 Order가 엔티티로 등록됨: true
[3] persist 성공, 생성된 id: 1
[4] find 성공, user_id: 42
```

메타모델 등록에서 그치지 않고 INSERT와 SELECT까지 확인한 이유는, 기동만 되고 컬럼 매핑이 어긋난 경우를 구분하기 위해서다.

> [blog-lab/002-java-entity-xml-only](https://github.com/BlueSF/blog-lab/tree/main/002-java-entity-xml-only)에서 `./verify.sh`로 재현할 수 있다. 인메모리 H2를 쓰므로 DB를 따로 띄울 필요는 없다.

Kotlin에서는 불가능하다. 같은 것을 하려 해도 `@Entity`가 남는다.

### 정리하면

```
Java   : 어노테이션 = 매핑 메타데이터                    → 클래스를 완전히 비울 수 있음
Kotlin : 어노테이션 = 매핑 메타데이터 + 컴파일러 스위치  → 클래스에 무언가는 반드시 남음
```

**"XML 하이브리드 매핑으로 도메인을 순수하게 만들겠다"는 목표는 Kotlin에서 절반까지만 달성된다.** 상세 매핑(테이블명·컬럼명·인덱스·컨버터 지정)은 전부 XML로 뺄 수 있지만, 엔티티 선언 자체는 클래스에 남는다.

---

## 남은 @Entity는 무엇인가

Kotlin에서 `@Entity`는 매핑 정보가 아니다. **컴파일러에게 이 클래스를 열고 생성자를 만들라고 지시하는 스위치**다. 지우고 싶어도 지울 수 있는 성격의 것이 아니다.

남은 `@Entity`는 인프라 정보가 아니라 빌드 지시문이다.

---

`@Entity`가 남는다는 건 확인했다. 그렇다면 나머지 어노테이션은 어떤가.

`@DynamicUpdate`, `@Converter`, `@CreatedDate`, `@Transient` — 이것들 중 무엇을 XML로 옮길 수 있고 무엇이 안 되는지는 [orm.xml로 옮길 수 있는 것과 없는 것](/posts/orm-xml-what-can-be-moved/)에서 다룬다. 미리 말해두면, 내가 "안 된다"고 믿었던 것 중 절반은 사실 **할 수 있는데 안 해놨던 것**이었다.
