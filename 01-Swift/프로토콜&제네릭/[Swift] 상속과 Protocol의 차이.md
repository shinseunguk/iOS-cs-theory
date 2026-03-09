# Swift 상속과 Protocol의 차이 정리

안녕하세요, 욱승입니다 👋

Swift 공부하다 보면 "상속이랑 프로토콜이 뭐가 다른 거지?" 하는 순간이 꼭 옵니다

둘 다 타입에 기능을 부여하는 방법인데, **설계 철학부터 내부 동작까지 완전히 다릅니다**

오늘은 이 둘을 비교하면서 언제 뭘 써야 하는지 정리해봤어요

---
<br>

## 1. 핵심 차이 한눈에 보기

| | 상속 (Inheritance) | 프로토콜 (Protocol) |
|:--|:--:|:--:|
| 사용 가능 타입 | `class`만 | `class`, `struct`, `enum` 전부 |
| 다중 채택 | ❌ 단일 상속만 | ✅ 여러 개 채택 가능 |
| 기본 구현 | 부모 클래스에서 제공 | `extension`으로 제공 |
| 저장 프로퍼티 | ✅ 상속 가능 | ❌ 요구만 가능 |
| 메서드 디스패치 | vtable (Dynamic) | PWT (Dynamic) 또는 Static |
| 관계 | "is-a" (Dog **is a** Animal) | "can-do" (Dog **can** Walkable) |
| 메모리 | Heap (참조 타입) | 타입에 따라 다름 |

---
<br>

## 2. 상속 (Inheritance)

### 개념

부모 클래스의 프로퍼티와 메서드를 **자식 클래스가 물려받는** 방식이에요
자식은 부모의 기능을 그대로 쓸 수도 있고, `override`로 재정의할 수도 있습니다

```swift
class Animal {
    var name: String

    init(name: String) {
        self.name = name
    }

    func speak() {
        print("...")
    }
}

class Dog: Animal {
    override func speak() {
        print("멍멍!")
    }
}

class Cat: Animal {
    override func speak() {
        print("야옹!")
    }
}

let dog = Dog(name: "초코")
dog.speak()  // "멍멍!"
dog.name     // "초코" ← 부모의 프로퍼티도 사용 가능
```

### 특징

- **단일 상속**: Swift에서 클래스는 부모를 하나만 가질 수 있어요
- **저장 프로퍼티 상속**: 부모의 저장 프로퍼티까지 물려받습니다
- **class 전용**: struct, enum은 상속이 안 돼요
- **vtable로 디스패치**: override된 메서드는 런타임에 vtable을 통해 찾습니다

```swift
// ❌ 다중 상속 불가
class Dog: Animal, Pet { }  // 컴파일 에러 (Pet이 class라면)
```

### 메모리 구조

상속하면 부모의 프로퍼티가 자식 인스턴스 안에 그대로 포함돼요

```
Dog 인스턴스 (Heap)
┌─────────────────────────────┐
│ metadata (vtable 포인터)     │
│ refCount                    │
│ name: "초코"    ← Animal 것 │
│ (Dog 고유 프로퍼티들...)     │
└─────────────────────────────┘
```

---
<br>

## 3. Protocol

### 개념

**"이런 기능을 구현해야 한다"는 약속(계약)** 이에요
실제 구현은 채택하는 타입이 직접 합니다

```swift
protocol Speakable {
    func speak()
}

protocol Walkable {
    func walk()
}

// struct도 채택 가능!
struct Dog: Speakable, Walkable {
    func speak() {
        print("멍멍!")
    }

    func walk() {
        print("총총총")
    }
}

// enum도 채택 가능!
enum Direction: Speakable {
    case north, south

    func speak() {
        print("방향: \(self)")
    }
}
```

### 특징

- **다중 채택**: 프로토콜은 여러 개 동시에 채택할 수 있어요
- **모든 타입 사용 가능**: class, struct, enum 전부 채택 가능
- **저장 프로퍼티 없음**: 요구사항만 정의하고 저장 프로퍼티는 못 가져요
- **기본 구현 제공 가능**: extension으로 기본 구현을 줄 수 있습니다

```swift
protocol Greetable {
    var name: String { get }
    func greet()
}

// extension으로 기본 구현 제공
extension Greetable {
    func greet() {
        print("안녕하세요, \(name)입니다!")
    }
}

struct User: Greetable {
    var name: String
    // greet()을 구현 안 해도 기본 구현이 있어서 OK
}

User(name: "욱승").greet()  // "안녕하세요, 욱승입니다!"
```

---
<br>

## 4. 같은 문제를 두 방식으로 풀어보기

"여러 도형을 그리는 기능"을 만든다고 해볼게요

### 상속으로 풀기

```swift
class Shape {
    func draw() {
        print("도형 그리기")
    }
}

class Circle: Shape {
    override func draw() {
        print("원 그리기")
    }
}

class Square: Shape {
    override func draw() {
        print("사각형 그리기")
    }
}

// 사용
let shapes: [Shape] = [Circle(), Square()]
shapes.forEach { $0.draw() }
```

문제점:
- struct로 만들 수 없어요 (Heap 할당 비용)
- 다른 기능(Resizable, Colorable 등)을 추가하려면 Shape에 계속 쌓아야 해요
- 단일 상속이라 구조가 경직됩니다

### 프로토콜로 풀기

```swift
protocol Drawable {
    func draw()
}

protocol Resizable {
    func resize(to scale: Double)
}

struct Circle: Drawable, Resizable {
    func draw() { print("원 그리기") }
    func resize(to scale: Double) { print("원 크기 변경: \(scale)") }
}

struct Square: Drawable {
    func draw() { print("사각형 그리기") }
}

// 사용
let drawables: [any Drawable] = [Circle(), Square()]
drawables.forEach { $0.draw() }
```

장점:
- struct 사용 가능 (Stack 할당, 성능 이점)
- 기능별로 프로토콜을 나눠서 필요한 것만 채택
- 유연한 조합이 가능합니다

---
<br>

## 5. 디스패치 방식 차이

이전 글에서 다뤘던 Dispatch 메커니즘이 여기서 연결돼요

### 상속 → vtable Dispatch

```swift
class Animal {
    func speak() { print("...") }  // vtable에 등록
}

class Dog: Animal {
    override func speak() { print("멍멍!") }  // vtable에서 포인터 교체
}

let animal: Animal = Dog()
animal.speak()  // 런타임에 vtable 조회 → Dog.speak 호출
```

### 프로토콜 → PWT 또는 Static Dispatch

```swift
protocol Speakable {
    func speak()
}

struct Dog: Speakable {
    func speak() { print("멍멍!") }
}

// ❌ any → PWT Dispatch (런타임, Existential Container 생성)
func makeSpeak1(_ animal: any Speakable) {
    animal.speak()
}

// ✅ some → Static Dispatch (컴파일 타임, 인라이닝 가능)
func makeSpeak2(_ animal: some Speakable) {
    animal.speak()
}

// ✅ Generic → Static Dispatch
func makeSpeak3<T: Speakable>(_ animal: T) {
    animal.speak()
}
```

프로토콜은 **어떻게 사용하느냐**에 따라 성능이 달라집니다
`some`이나 Generic을 쓰면 static dispatch가 돼서 상속보다 오히려 빨라요

---
<br>

## 6. 프로토콜 조합의 힘

상속으로는 불가능한, 프로토콜만의 강력한 패턴들이 있어요

### 프로토콜 합성 (Composition)

```swift
protocol Identifiable {
    var id: String { get }
}

protocol Displayable {
    var displayName: String { get }
}

// 두 프로토콜을 동시에 요구
func showInfo(_ item: some Identifiable & Displayable) {
    print("[\(item.id)] \(item.displayName)")
}
```

### 프로토콜 + 제네릭 제약

```swift
protocol Cacheable {
    var cacheKey: String { get }
}

// Cacheable을 채택한 Codable 타입만 받겠다
func save<T: Cacheable & Codable>(_ item: T) {
    let data = try? JSONEncoder().encode(item)
    // cacheKey로 저장...
}
```

### where절로 세밀한 제약

```swift
protocol Container {
    associatedtype Item
    var items: [Item] { get }
}

extension Container where Item: Equatable {
    func contains(_ item: Item) -> Bool {
        items.contains(item)
    }
}

extension Container where Item == String {
    func joinAll() -> String {
        items.joined(separator: ", ")
    }
}
```

상속으로는 이런 유연한 타입 제약이 불가능합니다

---
<br>

## 7. 언제 뭘 써야 하냐면

### 상속이 적합한 경우

- UIKit 같은 프레임워크가 class 상속을 요구할 때
- 부모의 **저장 프로퍼티와 구현을 그대로** 물려받아야 할 때
- 객체 간에 명확한 **is-a 관계**가 있을 때

```swift
// UIKit은 상속 기반
class MyViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()
    }
}
```

### 프로토콜이 적합한 경우

- **기능 단위**로 타입을 설계할 때
- struct/enum에도 공통 인터페이스를 적용하고 싶을 때
- 여러 기능을 **조합**해서 쓰고 싶을 때
- 테스트를 위한 **mock 객체**를 만들 때

```swift
// 프로토콜로 의존성 주입 → 테스트 용이
protocol NetworkServiceProtocol {
    func fetch(url: String) async throws -> Data
}

struct RealNetworkService: NetworkServiceProtocol {
    func fetch(url: String) async throws -> Data { ... }
}

struct MockNetworkService: NetworkServiceProtocol {
    func fetch(url: String) async throws -> Data {
        return testData  // 테스트용 가짜 데이터
    }
}
```

---
<br>

## 8. 한눈에 정리

```
타입에 기능을 부여하고 싶다
    │
    ├─ 부모의 저장 프로퍼티/구현을 물려받아야 한다?
    │       └─ 상속 (class only, 단일 상속)
    │
    ├─ struct/enum에서도 쓰고 싶다?
    │       └─ 프로토콜
    │
    ├─ 여러 기능을 조합하고 싶다?
    │       └─ 프로토콜 (다중 채택 가능)
    │
    └─ 성능이 중요하다?
            └─ 프로토콜 + some/Generic (Static Dispatch)
```

---
<br>

## 결론

Swift는 공식적으로 **Protocol-Oriented Programming(POP)** 을 지향하고 있어요

상속은 UIKit처럼 프레임워크가 요구하거나, 진짜 is-a 관계가 명확할 때 쓰고
그 외에는 프로토콜로 설계하는 게 더 유연하고 성능도 좋습니다

이전 글에서 다뤘던 Dispatch 메커니즘이랑 연결해서 보면
왜 Apple이 프로토콜을 강조하는지 더 납득이 될 거에요 🫡

문의는 댓글!

---

Ref.
- [Apple Developer - Protocols](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/protocols/)
- [WWDC 2015 - Protocol-Oriented Programming in Swift](https://developer.apple.com/videos/play/wwdc2015/408/)
