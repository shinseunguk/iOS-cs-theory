# Swift Dispatch 메커니즘 정리 - Static vs Dynamic vs Protocol Witness Table

안녕하세요, 욱승입니다 👋

이전 글에서 Struct vs Class 메모리 구조랑 ARC 동작 방식을 다뤘는데요

오늘은 **메서드를 호출할 때 Swift가 내부적으로 어떻게 함수를 찾아서 실행하는지** 정리해봤습니다

면접에서 "Static Dispatch와 Dynamic Dispatch 차이가 뭔가요?" 이런 질문이 나오면
이 글 하나로 정리할 수 있을 거에요

---
<br>

## 1. 개요

Swift에서 메서드 호출 방식은 크게 세 가지로 나뉩니다

| 방식 | 결정 시점 | 속도 | 사용 조건 |
|------|-----------|------|-----------|
| **Static Dispatch** | 컴파일 타임 | 가장 빠름 | `struct`, `enum`, `final class`, `static/class final` |
| **Dynamic Dispatch (vtable)** | 런타임 | 보통 | 클래스 상속, `override` 가능한 메서드 |
| **Dynamic Dispatch (PWT)** | 런타임 | 보통 | `any Protocol` (Existential) |

하나씩 살펴볼게요

---
<br>

## 2. Static Dispatch

### 개념

컴파일러가 **어떤 함수를 호출할지 컴파일 타임에 결정**하고, 해당 함수 주소를 직접 코드에 삽입하는 방식이에요

런타임에 "이거 어떤 함수지?" 하고 찾아볼 필요가 없으니까 가장 빠릅니다

### 발생 조건

```swift
// 1. struct / enum 의 메서드
struct Circle {
    func draw() { print("Circle") } // → static dispatch
}

// 2. final class
final class Logger {
    func log() { print("log") }    // → static dispatch
}

// 3. 일반 class에 final 붙인 메서드
class ViewController {
    final func setup() {}          // → static dispatch
}

// 4. Generic (타입 특수화 후)
func render(_ shape: T) {
    shape.draw()                   // → T가 확정되면 static dispatch
}
```

struct, enum은 상속이 안 되니까 메서드가 바뀔 일이 없어요
그래서 컴파일러가 "이 함수 호출하면 됨" 하고 바로 주소를 박아넣을 수 있는 거에요

### 컴파일러 최적화 (Inlining)

Static dispatch 덕분에 컴파일러는 함수 자체를 **호출 지점에 인라이닝**할 수 있습니다

```swift
// 원본
let c = Circle()
c.draw()

// 컴파일러가 실제로 생성하는 것 (개념적)
print("Circle")  // 함수 호출 없이 본문을 직접 삽입
```

함수 호출 자체를 없애버리니까 성능이 더 좋아지는 거에요

---
<br>

## 3. Dynamic Dispatch — vtable (클래스 상속)

### 개념

클래스 상속을 쓰면 서브클래스가 메서드를 `override`할 수 있잖아요
그러면 컴파일러 입장에서는 "이 변수가 실제로 어떤 타입인지" 런타임에 가봐야 알아요

그래서 각 클래스는 자신의 메서드 포인터 목록인 **vtable(Virtual Method Table)** 을 갖고 있습니다

### 구조

```
Animal 클래스
┌─────────────────────────────────┐
│ vtable                          │
│   speak → Animal.speak 포인터   │
└─────────────────────────────────┘

Dog 클래스 (Animal 상속)
┌─────────────────────────────────┐
│ vtable                          │
│   speak → Dog.speak 포인터      │  ← override로 교체됨
└─────────────────────────────────┘
```

Dog가 speak을 override하면 Dog의 vtable에서 speak 포인터가 Dog.speak으로 교체되는 거에요

### 동작 흐름

```swift
class Animal {
    func speak() { print("...") }
}

class Dog: Animal {
    override func speak() { print("Woof") }
}

let animal: Animal = Dog()
animal.speak()
// 1. animal 객체 → 타입 메타데이터 접근
// 2. 메타데이터 → vtable 조회
// 3. vtable[speak] → Dog.speak 포인터
// 4. 호출
```

타입이 `Animal`로 선언돼 있어도 실제 인스턴스가 `Dog`니까 Dog의 vtable을 뒤져서 Dog.speak을 호출하는 거에요

### 호출 비용

```
일반 함수 호출:  주소 직접 점프
vtable 호출:    포인터 역참조(1회) → 조회 → 점프
```

포인터 역참조 한 번이 추가될 뿐이라 실제 오버헤드는 미미합니다
근데 **인라이닝이 불가능**해서 반복 호출이 많으면 누적 비용이 생길 수 있어요

---
<br>

## 4. Dynamic Dispatch — Protocol Witness Table (PWT)

### 개념

여기가 좀 재밌는 부분이에요

`any Protocol` (Existential 타입)으로 값을 다룰 때 사용하는 방식입니다
프로토콜을 채택한 **각 타입마다 PWT가 하나씩 생성**되고, 해당 타입의 프로토콜 메서드 구현 포인터들을 담아요

### 구조

```
Circle의 Drawable PWT
┌────────────────────────────────────┐
│ draw  → Circle.draw 함수 포인터    │
└────────────────────────────────────┘

Square의 Drawable PWT
┌────────────────────────────────────┐
│ draw  → Square.draw 함수 포인터    │
└────────────────────────────────────┘
```

vtable이랑 비슷해 보이지만 좀 다릅니다
vtable은 클래스 메타데이터에 붙어있고, PWT는 **타입 × 프로토콜 조합**마다 별도로 존재해요

### Existential Container

`any Drawable`로 값을 전달할 때 Swift는 내부적으로 **5 word짜리 Existential Container**를 만듭니다
이게 은근 커요

```
Existential Container (40 bytes on 64-bit)
┌──────────────────────────────────────┐
│ value buffer [0] (8 bytes)           │
│ value buffer [1] (8 bytes)           │  ← 작은 값은 직접 저장
│ value buffer [2] (8 bytes)           │  ← 큰 값은 heap 포인터
│ value witness table pointer (8 bytes)│  ← 값 복사/소멸 방법
│ protocol witness table pointer(8 bytes)│← 메서드 디스패치
└──────────────────────────────────────┘
```

value buffer가 3 word(24 bytes)인데, 값이 여기에 안 들어가면 **heap 할당**이 일어나요
struct가 프로퍼티 많으면 이거 때문에 성능 손해를 볼 수 있습니다

### 동작 흐름

```swift
protocol Drawable {
    func draw()
}

func render(_ shape: any Drawable) {
    shape.draw()
    // 1. Existential Container에서 PWT 포인터 꺼냄
    // 2. PWT에서 draw 함수 포인터 찾음
    // 3. value buffer에서 실제 값 꺼냄
    // 4. 호출
}
```

vtable보다 한 단계가 더 있는 셈이에요

### vtable과의 차이점

| | vtable | PWT |
|---|---|---|
| 대상 | 클래스 상속 | 프로토콜 채택 |
| 생성 단위 | 클래스당 1개 | **타입 × 프로토콜 조합**당 1개 |
| 테이블 위치 | 타입 메타데이터 내부 | 별도 테이블 |
| 추가 비용 | 포인터 역참조 1회 | Existential Container 생성 + 포인터 역참조 |
| 힙 할당 | 없음 (참조 타입이므로) | 값이 3 word 초과 시 heap 할당 |

---
<br>

## 5. 세 방식 비교 및 선택 기준

```swift
protocol Drawable {
    func draw()
}

struct Circle: Drawable {
    func draw() { print("Circle") }
}

// ❌ Dynamic (PWT) — Existential Container 생성, heap 할당 가능
func render1(_ shape: any Drawable) {
    shape.draw()
}

// ✅ Static — Generic specialization으로 컴파일 타임 결정
func render2(_ shape: T) {
    shape.draw()
}

// ✅ Static (Swift 5.7+) — some을 쓰면 Opaque Type으로 static dispatch
func render3(_ shape: some Drawable) {
    shape.draw()
}
```

`any`로 받으면 PWT 거치고, `some`이나 Generic으로 받으면 static dispatch가 됩니다
같은 프로토콜을 쓰더라도 **어떻게 받느냐**에 따라 성능이 달라지는 거에요

### 성능 영향 요소

```
any Protocol 사용 시 비용:
1. Existential Container 스택 할당 (40 bytes)
2. 값이 크면 heap 할당 (malloc)
3. PWT 포인터 역참조
4. 인라이닝 불가

Generic / some 사용 시:
1. 컴파일 타임 특수화
2. 인라이닝 가능
3. 런타임 오버헤드 없음
```

이거 보면 왜 `some`이랑 Generic을 권장하는지 이해가 되죠

---
<br>

## 6. 최적화 전략

### 1) `any` → `some` 또는 Generic으로 교체

```swift
// Before
func process(_ items: [any Drawable]) { ... }

// After
func process(_ items: [T]) { ... }
// 또는
func process(_ item: some Drawable) { ... }
```

이것만 바꿔도 PWT 비용이 사라집니다

### 2) 클래스에 `final` 선언

```swift
// vtable dispatch
class MyService {
    func fetch() { ... }
}

// static dispatch
final class MyService {
    func fetch() { ... }
}
```

상속 안 할 클래스면 `final` 붙여주는 게 좋아요
vtable 조회가 없어지니까요

### 3) WMO (Whole Module Optimization) 활성화

모듈 전체를 한 번에 컴파일해서 `final`을 추론하고 더 많은 static dispatch를 적용합니다

```
Xcode Build Settings
→ Compilation Mode: Whole Module
→ Optimization Level: Optimize for Speed [-O]
```

### 4) `@inlinable`로 모듈 경계 넘어 인라이닝

```swift
@inlinable
public func draw(_ shape: some Drawable) {
    shape.draw()  // 외부 모듈에서도 인라이닝 가능
}
```

프레임워크 만들 때 유용한 방법이에요

---
<br>

## 7. 전체 흐름 요약

```
메서드 호출
    │
    ├─ struct / enum / final class?
    │       └─ Static Dispatch ✅ (컴파일 타임, 인라이닝 가능)
    │
    ├─ class (override 가능)?
    │       └─ vtable Dispatch ⚡ (런타임, 포인터 역참조 1회)
    │
    └─ any Protocol (Existential)?
            └─ PWT Dispatch ⚠️ (런타임, Container 생성, heap 할당 가능)

최적화 방향: any → some / Generic → final → @inlinable
```

---
<br>

## 결론

정리하면 Swift에서 메서드 호출은 **타입이 뭐냐, 어떻게 선언했냐**에 따라 dispatch 방식이 달라집니다

- struct/enum/final class → **Static Dispatch** (가장 빠름)
- 클래스 상속 → **vtable** (포인터 한 번 더 거침)
- any Protocol → **PWT** (Container 생성 + heap 할당 가능성)

성능이 중요한 곳에서는 `any` 대신 `some`이나 Generic을 쓰고, 클래스에는 `final`을 붙이는 습관을 들이면 좋습니다

이전 글에서 다뤘던 Struct vs Class 메모리 구조, ARC 동작 방식이랑 같이 보면 더 이해가 잘 될 거에요 🫡

문의는 댓글!

---

Ref.
- [Apple Developer - Method Dispatch](https://developer.apple.com/swift/)
- [Swift Forum - Understanding Swift Performance](https://forums.swift.org/)
