# Swift associatedtype 동작 원리

안녕하세요, 욱승입니다 👋

오늘은 **associatedtype의 동작 원리**를 정리해봤습니다

프로토콜에서 제네릭처럼 타입을 추상화하는 핵심 도구인데, 내부적으로 어떻게 동작하는지 깊이 파보겠습니다

---
<br>

## 1. 기본 개념

`associatedtype`은 **프로토콜 내에서 사용되는 플레이스홀더 타입**으로, 프로토콜을 채택하는 타입이 구체적인 타입을 결정합니다

```swift
protocol Container {
    associatedtype Item  // 아직 구체 타입이 정해지지 않음
    func append(_ item: Item)
    func get(at index: Int) -> Item
}
```

채택하는 쪽에서 `Item`이 무엇인지 결정합니다

```swift
struct IntStack: Container {
    typealias Item = Int  // 명시적 지정 (생략 가능)

    func append(_ item: Int) { ... }
    func get(at index: Int) -> Int { ... }
}

struct StringQueue: Container {
    // typealias 생략 — 컴파일러가 메서드 시그니처에서 Item = String으로 추론
    func append(_ item: String) { ... }
    func get(at index: Int) -> String { ... }
}
```

---
<br>

## 2. 제네릭과의 차이

둘 다 타입을 추상화하지만 **타입 결정 주체**가 다릅니다

```swift
// Generic — 호출하는 쪽이 타입을 결정
func swapValues<T>(_ a: inout T, _ b: inout T) { ... }

// associatedtype — 채택하는 쪽이 타입을 결정
protocol Iterator {
    associatedtype Element
    func next() -> Element?
}
```

| | Generic | associatedtype |
|---|---------|---------------|
| 타입 결정 | 호출자 | 채택자 (구현체) |
| 사용 위치 | 함수, 구조체, 클래스 | 프로토콜 전용 |
| 타입 관계 | 독립적 | 프로토콜 내 메서드 간 타입 일관성 보장 |

프로토콜은 `<T>` 문법을 쓸 수 없기 때문에, **associatedtype이 "프로토콜의 제네릭"** 역할을 합니다

---
<br>

## 3. 타입 추론 과정 (컴파일러 동작)

컴파일러는 **메서드 시그니처에서 associatedtype을 역추론**합니다

```swift
protocol Converter {
    associatedtype Input
    associatedtype Output
    func convert(_ input: Input) -> Output
}

struct StringToInt: Converter {
    // 컴파일러가 추론: Input = String, Output = Int
    func convert(_ input: String) -> Int {
        return input.count
    }
}
```

추론 순서:

1. 프로토콜 요구 메서드의 시그니처를 확인
2. 구현체의 실제 메서드 시그니처와 매칭
3. `Input → String`, `Output → Int`로 결정
4. 프로토콜 내 모든 곳에서 해당 타입을 일관되게 적용

따라서 `typealias`는 컴파일러가 추론할 수 없는 경우에만 명시하면 됩니다

---
<br>

## 4. 제약 조건 (where)

associatedtype에 제약을 걸어서 채택자가 사용할 수 있는 타입을 제한할 수 있어요

```swift
protocol SortableContainer {
    associatedtype Item: Comparable  // Item은 반드시 Comparable이어야 함
    var items: [Item] { get }
    func sorted() -> [Item]
}

// 더 복잡한 제약
protocol Repository {
    associatedtype Entity: Identifiable where Entity.ID == UUID
    func find(by id: UUID) -> Entity?
}
```

프로토콜 확장에서도 where절을 활용할 수 있습니다

```swift
extension Container where Item: Equatable {
    func contains(_ item: Item) -> Bool {
        // Item이 Equatable일 때만 이 메서드 사용 가능
        for i in 0..<count {
            if get(at: i) == item { return true }
        }
        return false
    }
}
```

---
<br>

## 5. PAT (Protocol with Associated Type) 제한

associatedtype이 있는 프로토콜은 **타입으로 직접 사용할 수 없습니다**

```swift
protocol Animal {
    associatedtype Food
    func eat(_ food: Food)
}

// ❌ 컴파일 에러 — Food가 뭔지 모르기 때문
func feedAnimal(_ animal: Animal) { }

// ✅ 해결 1: 제네릭으로
func feedAnimal<A: Animal>(_ animal: A) { }

// ✅ 해결 2: some (Opaque Type, Swift 5.1+)
func feedAnimal(_ animal: some Animal) { }

// ✅ 해결 3: any (Existential, Swift 5.7+)
func feedAnimal(_ animal: any Animal) { }  // 제한적 사용
```

**왜 안 되는가?**

컴파일러가 `Food`의 구체 타입을 알 수 없으면 메모리 레이아웃과 메서드 디스패치를 결정할 수 없기 때문입니다

`Animal`이라는 프로토콜만으로는 `eat()` 메서드에 어떤 타입의 인자를 넘겨야 하는지 알 수 없어요

---
<br>

## 6. some vs any 로 PAT 사용

```swift
// some — 컴파일 타임에 구체 타입 고정 (더 빠름)
func getAnimal() -> some Animal {
    return Dog()  // 항상 같은 타입만 반환 가능
}

// any — 런타임에 다양한 타입 허용 (Existential Container 사용)
func getAnimals() -> [any Animal] {
    return [Dog(), Cat()]  // 서로 다른 타입 가능
}
```

| | some | any |
|---|------|-----|
| 타입 결정 | 컴파일 타임 | 런타임 |
| 성능 | Static Dispatch (빠름) | Dynamic Dispatch (느림) |
| 다른 타입 섞기 | ❌ | ✅ |
| associatedtype 접근 | ✅ 자유롭게 | ❌ 제한적 |

---
<br>

## 7. 실무 예시 — Swift 표준 라이브러리

Swift 표준 라이브러리의 핵심 프로토콜들이 모두 associatedtype을 사용합니다

```swift
// Collection 프로토콜 (단순화)
protocol Collection {
    associatedtype Element
    associatedtype Index: Comparable

    var startIndex: Index { get }
    var endIndex: Index { get }
    subscript(position: Index) -> Element { get }
}

// Array: Element = 제네릭 T, Index = Int
// Dictionary: Element = (Key, Value), Index = Dictionary.Index
// String: Element = Character, Index = String.Index
```

하나의 프로토콜로 다양한 컬렉션 타입을 통일된 인터페이스로 다룰 수 있는 이유가 바로 associatedtype 덕분이에요

---
<br>

## 면접 포인트 정리

- **associatedtype은 "프로토콜의 제네릭"**: 프로토콜은 `<T>` 문법을 쓸 수 없어서 associatedtype으로 대체
- **타입 추론**: `typealias` 없이도 메서드 시그니처에서 자동 추론됨
- **PAT 제한 이유**: 컴파일러가 구체 타입을 모르면 메모리 레이아웃을 결정 불가
- **some vs any**: `some`은 컴파일 타임 고정(Static Dispatch, 성능 좋음), `any`는 런타임 유연성(Existential Container 오버헤드)
- **where 제약**: associatedtype에 프로토콜 제약이나 타입 동등 제약을 걸어서 타입 안전성 확보
