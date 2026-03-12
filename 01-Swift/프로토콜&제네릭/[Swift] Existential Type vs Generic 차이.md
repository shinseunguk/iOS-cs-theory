# Swift Existential Type vs Generic 차이 정리

안녕하세요, 욱승입니다 👋

Swift에서 프로토콜을 사용하는 방식은 크게 두 가지예요

**Existential Type (`any`)** 과 **Generic (`some`, `<T>`)** 인데, 겉보기엔 비슷해 보여도 **내부 동작과 성능이 완전히 다릅니다**

오늘은 이 둘의 차이를 깊이 파헤쳐볼게요

---
<br>

## 1. 핵심 차이 한눈에 보기

| | Existential (`any`) | Generic (`some` / `<T>`) |
|:--|:--:|:--:|
| 타입 결정 시점 | 런타임 | 컴파일 타임 |
| 디스패치 | Dynamic (PWT) | Static (인라이닝 가능) |
| 메모리 | Existential Container (최대 40바이트) | 타입 크기 그대로 |
| 다른 타입 섞기 | ✅ 가능 | ❌ 불가 |
| associatedtype 사용 | ❌ 제한적 (Swift 5.7+부터 일부 가능) | ✅ 자유롭게 사용 |
| 성능 | 상대적으로 느림 | 빠름 (최적화 유리) |

---
<br>

## 2. Existential Type (`any`)

### 개념

**"이 프로토콜을 채택한 어떤 타입이든 담을 수 있는 박스"** 예요
런타임에 실제 타입이 결정되기 때문에 **타입 소거(Type Erasure)** 가 일어납니다

```swift
protocol Animal {
    func speak()
}

struct Dog: Animal {
    func speak() { print("멍멍!") }
}

struct Cat: Animal {
    func speak() { print("야옹!") }
}

// any → 서로 다른 타입을 하나의 배열에 담을 수 있음
let animals: [any Animal] = [Dog(), Cat()]
animals.forEach { $0.speak() }
```

### Existential Container 구조

`any`를 쓰면 내부적으로 **Existential Container** 라는 박스가 만들어져요

```
Existential Container (40바이트)
┌──────────────────────────────────┐
│ Value Buffer (24바이트)           │  ← 작은 타입은 여기에 인라인 저장
│                                  │  ← 큰 타입은 Heap 할당 후 포인터 저장
├──────────────────────────────────┤
│ Value Witness Table 포인터 (8바이트) │  ← 메모리 관리 (copy, destroy 등)
├──────────────────────────────────┤
│ Protocol Witness Table 포인터 (8바이트) │  ← 메서드 디스패치
└──────────────────────────────────┘
```

**Value Buffer는 24바이트**인데:
- 타입 크기가 24바이트 이하면 → Value Buffer 안에 직접 저장 (인라인)
- 타입 크기가 24바이트 초과면 → Heap에 할당하고 포인터만 저장

```swift
// 24바이트 이하 → 인라인 저장 ✅
struct SmallDog: Animal {
    var x: Int  // 8바이트
    var y: Int  // 8바이트
    func speak() { print("멍!") }
}

// 24바이트 초과 → Heap 할당 ❌
struct BigDog: Animal {
    var a: Int  // 8
    var b: Int  // 8
    var c: Int  // 8
    var d: Int  // 8 → 총 32바이트
    func speak() { print("왈왈!") }
}
```

### 성능 비용

```swift
func makeSpeak(_ animal: any Animal) {
    animal.speak()
}
```

이 코드가 실행될 때 벌어지는 일:
1. Existential Container에서 PWT 포인터를 꺼냄
2. PWT에서 `speak` 메서드의 함수 포인터를 찾음
3. Value Buffer에서 실제 값을 꺼냄 (또는 Heap 포인터 역참조)
4. 함수 호출

**매번 간접 참조가 발생**하고, 컴파일러가 인라이닝을 못 해요

---
<br>

## 3. Generic (`some` / `<T>`)

### 개념

**"컴파일 타임에 구체 타입이 결정되는 방식"** 이에요
컴파일러가 실제 타입을 알기 때문에 **최적화가 가능**합니다

```swift
// Generic 문법
func makeSpeak<T: Animal>(_ animal: T) {
    animal.speak()
}

// some 문법 (Swift 5.7+, 위와 동일한 의미)
func makeSpeak(_ animal: some Animal) {
    animal.speak()
}
```

### 단형성화 (Monomorphization)

컴파일러가 Generic 함수를 호출하는 각 구체 타입마다 **전용 함수를 생성**해요

```swift
makeSpeak(Dog())   // → 컴파일러가 makeSpeak_Dog() 생성
makeSpeak(Cat())   // → 컴파일러가 makeSpeak_Cat() 생성
```

이렇게 되면:
- 타입이 확정되므로 **Static Dispatch** 가능
- 함수 본문을 **인라이닝** 할 수 있음
- Existential Container **불필요**
- 결과적으로 일반 함수 호출과 동일한 성능

### `some` vs `<T>` 차이

둘 다 Generic이지만 미묘한 차이가 있어요

```swift
// some → 호출자가 타입을 결정, 함수 내부에서 타입 이름 사용 불가
func feed(_ animal: some Animal) {
    animal.speak()  // OK
    // T를 참조할 방법이 없음
}

// <T> → 타입 매개변수를 명시적으로 사용 가능
func feed<T: Animal>(_ animal: T) -> T {
    // T를 반환 타입에 쓸 수 있음
    return animal
}

// <T> → 같은 타입을 여러 매개변수에 강제 가능
func compare<T: Animal>(_ a: T, _ b: T) {
    // a와 b는 반드시 같은 타입
}

compare(Dog(), Dog())  // ✅ OK
compare(Dog(), Cat())  // ❌ 컴파일 에러
```

**정리하면:**
- 단순히 "프로토콜 준수하는 타입 하나 받을래" → `some`
- 타입 매개변수가 필요하거나 복잡한 제약 → `<T>`

---
<br>

## 4. 같은 코드를 두 방식으로 비교

### Existential로 풀기

```swift
protocol Drawable {
    func draw()
}

struct Circle: Drawable {
    func draw() { print("원") }
}

struct Square: Drawable {
    func draw() { print("사각형") }
}

// ✅ 서로 다른 타입을 섞을 수 있음
let shapes: [any Drawable] = [Circle(), Square()]

// ❌ 매번 PWT 조회 → Dynamic Dispatch
func drawAll(_ shapes: [any Drawable]) {
    for shape in shapes {
        shape.draw()
    }
}
```

### Generic으로 풀기

```swift
// ❌ 한 가지 타입만 가능
let circles: [some Drawable] = [Circle(), Circle()]
// [Circle(), Square()] → 컴파일 에러!

// ✅ Static Dispatch → 인라이닝 가능
func drawAll<T: Drawable>(_ shapes: [T]) {
    for shape in shapes {
        shape.draw()
    }
}

drawAll([Circle(), Circle()])  // ✅
drawAll([Circle(), Square()])  // ❌ 타입이 다름
```

**핵심:** 유연성(다른 타입 섞기)이 필요하면 `any`, 성능이 중요하면 Generic

---
<br>

## 5. associatedtype과의 관계

Generic과 Existential의 가장 큰 실용적 차이가 여기서 나옵니다

```swift
protocol Container {
    associatedtype Item
    var items: [Item] { get }
    mutating func add(_ item: Item)
}

struct IntBox: Container {
    var items: [Int] = []
    mutating func add(_ item: Int) {
        items.append(item)
    }
}

struct StringBox: Container {
    var items: [String] = []
    mutating func add(_ item: String) {
        items.append(item)
    }
}
```

### Generic → 문제없이 동작

```swift
func printItems<T: Container>(_ container: T) where T.Item: CustomStringConvertible {
    for item in container.items {
        print(item)
    }
}

printItems(IntBox(items: [1, 2, 3]))       // ✅
printItems(StringBox(items: ["a", "b"]))   // ✅
```

### Existential → 제한적

```swift
// Swift 5.7+ 에서 any로 사용 가능하지만 제약이 있음
func printItems(_ container: any Container) {
    // container.items → [Any]로 타입 소거됨
    // container.add(???) → Item 타입을 모르니 호출 어려움
}

// ❌ 이런 건 안 됨
let containers: [any Container] = [IntBox(), StringBox()]
for c in containers {
    c.add(???)  // Item이 Int인지 String인지 모름
}
```

**associatedtype이 있는 프로토콜**은 Generic으로 쓰는 게 자연스럽습니다

---
<br>

## 6. 성능 벤치마크 비교

실제로 얼마나 차이 나는지 대략적인 비교에요

```swift
protocol Calculatable {
    func calculate() -> Int
}

struct Calculator: Calculatable {
    var value: Int
    func calculate() -> Int { value * 2 }
}

// Existential 방식
func sumExistential(_ items: [any Calculatable]) -> Int {
    var result = 0
    for item in items {
        result += item.calculate()  // PWT 조회 매번 발생
    }
    return result
}

// Generic 방식
func sumGeneric<T: Calculatable>(_ items: [T]) -> Int {
    var result = 0
    for item in items {
        result += item.calculate()  // 인라이닝 → value * 2가 직접 삽입
    }
    return result
}
```

```
100만 번 반복 기준 (대략적):
┌─────────────────────┬──────────┐
│ 방식                 │ 상대 속도 │
├─────────────────────┼──────────┤
│ Generic (some / <T>) │ 1x (기준) │
│ Existential (any)    │ 2~5x 느림 │
└─────────────────────┴──────────┘
```

차이가 나는 이유:
- Generic → 컴파일러가 `value * 2`를 직접 인라이닝
- Existential → 매번 Container에서 값 꺼내기 + PWT 조회 + 간접 호출

---
<br>

## 7. 실전 사용 가이드

### `any`를 써야 할 때

```swift
// 1. 서로 다른 타입을 컬렉션에 담아야 할 때
let animals: [any Animal] = [Dog(), Cat(), Bird()]

// 2. 프로퍼티로 다양한 타입을 저장해야 할 때
class ViewController {
    var delegate: (any ViewControllerDelegate)?
}

// 3. 타입이 런타임에 결정되는 팩토리 패턴
func createAnimal(type: String) -> any Animal {
    switch type {
    case "dog": return Dog()
    case "cat": return Cat()
    default: fatalError()
    }
}
```

### `some` / Generic을 써야 할 때

```swift
// 1. 함수 매개변수로 프로토콜 타입을 받을 때 (기본값)
func feed(_ animal: some Animal) {
    animal.speak()
}

// 2. associatedtype이 있는 프로토콜
func process<C: Collection>(_ collection: C) where C.Element: Equatable {
    // ...
}

// 3. SwiftUI의 body처럼 Opaque Return Type
var body: some View {
    Text("Hello")
}

// 4. 같은 타입을 보장해야 할 때
func swap<T: Animal>(_ a: inout T, _ b: inout T) {
    let temp = a
    a = b
    b = temp
}
```

---
<br>

## 8. 한눈에 정리

```
프로토콜을 타입으로 사용하고 싶다
    │
    ├─ 서로 다른 타입을 섞어야 한다?
    │       └─ any (Existential) → 런타임, 유연하지만 느림
    │
    ├─ 한 가지 타입만 쓰는데 추상화하고 싶다?
    │       └─ some / Generic → 컴파일 타임, 빠름
    │
    ├─ associatedtype이 있다?
    │       └─ Generic 우선 → any는 제약이 많음
    │
    └─ 성능이 중요하다?
            └─ some / Generic → Static Dispatch + 인라이닝
```

---
<br>

## 결론

Swift 5.7부터 `any` 키워드가 도입되면서 Existential과 Generic의 구분이 더 명확해졌어요

**기본적으로 `some` / Generic을 쓰고, 정말 다른 타입을 섞어야 할 때만 `any`를 쓰는 게 정석**입니다

이전 글에서 다뤘던 PWT, Static/Dynamic Dispatch와 직접 연결되는 내용이니까
같이 보면 왜 Apple이 이런 설계를 했는지 더 잘 이해될 거에요 🫡

문의는 댓글!

---

Ref.
- [Swift Documentation - Opaque and Boxed Types](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/opaquetypes/)
- [WWDC 2022 - Embrace Swift Generics](https://developer.apple.com/videos/play/wwdc2022/110352/)
- [SE-0335 - Introduce existential `any`](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0335-existential-any.md)
