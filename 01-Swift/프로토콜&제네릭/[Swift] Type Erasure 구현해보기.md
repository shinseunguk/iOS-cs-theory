# Swift Type Erasure 구현해보기

안녕하세요, 욱승입니다 👋

오늘은 **Type Erasure(타입 소거)** 를 직접 구현해보겠습니다

associatedtype이 있는 프로토콜(PAT)은 타입으로 직접 사용할 수 없는데, Type Erasure가 이 문제를 어떻게 해결하는지 단계별로 알아볼게요

---
<br>

## 1. 문제 상황 — PAT는 타입으로 못 쓴다

```swift
protocol Animal {
    associatedtype Food
    func eat(_ food: Food)
}

struct Dog: Animal {
    func eat(_ food: String) { print("🐕 \(food) 먹는 중") }
}

struct Cat: Animal {
    func eat(_ food: Int) { print("🐱 사료 \(food)g 먹는 중") }
}

// ❌ 컴파일 에러 — "Use of protocol 'Animal' as a type must be written 'any Animal'"
// 그리고 any Animal로 써도 Food를 모르니 eat()을 호출할 수 없음
let animals: [Animal] = [Dog(), Cat()]
```

**Dog의 Food는 String, Cat의 Food는 Int** — 타입이 다르니 하나의 배열에 담을 수 없어요

같은 Food 타입을 가진 Animal끼리는 묶고 싶은데, 프로토콜 자체를 타입으로 쓸 수 없으니 막힙니다

---
<br>

## 2. Type Erasure란?

**구체 타입 정보를 지우고(erase), 프로토콜 인터페이스만 남기는 래퍼 타입**입니다

Swift 표준 라이브러리의 `AnySequence`, `AnyPublisher`, `AnyHashable` 등이 전부 Type Erasure 패턴이에요

핵심 아이디어:

```
구체 타입(Dog, Cat) → 래퍼(AnyAnimal)로 감싸기 → 외부에서는 구체 타입을 모름
```

---
<br>

## 3. 단계별 구현

### Step 1: 프로토콜 정의

```swift
protocol Animal {
    associatedtype Food
    func eat(_ food: Food)
    var name: String { get }
}

struct Dog: Animal {
    typealias Food = String
    let name = "강아지"
    func eat(_ food: String) { print("\(name)가 \(food) 먹는 중") }
}

struct Cat: Animal {
    typealias Food = String
    let name = "고양이"
    func eat(_ food: String) { print("\(name)가 \(food) 먹는 중") }
}
```

Dog과 Cat 모두 Food가 String인 상황입니다

### Step 2: 내부 박스 (타입을 지우는 핵심)

```swift
// 추상 박스 — Food 타입만 알고, 구체 타입은 모름
private class _AnyAnimalBox<Food> {
    func eat(_ food: Food) { fatalError("override 필요") }
    var name: String { fatalError("override 필요") }
}

// 구체 박스 — 실제 Animal 구현체를 감싸서 보관
private class _AnimalBox<A: Animal>: _AnyAnimalBox<A.Food> {
    let base: A

    init(_ base: A) { self.base = base }

    override func eat(_ food: A.Food) { base.eat(food) }
    override var name: String { base.name }
}
```

여기서 **타입 소거가 발생**합니다

- `_AnimalBox<Dog>`은 `_AnyAnimalBox<String>`을 상속
- `_AnimalBox<Cat>`도 `_AnyAnimalBox<String>`을 상속
- 부모 타입인 `_AnyAnimalBox<String>`으로 보면 Dog인지 Cat인지 구분 불가 → **타입이 지워짐**

### Step 3: 외부 래퍼 (AnyAnimal)

```swift
struct AnyAnimal<Food>: Animal {
    private let box: _AnyAnimalBox<Food>

    // 제네릭 이니셜라이저 — 어떤 Animal이든 받되, Food 타입이 같아야 함
    init<A: Animal>(_ animal: A) where A.Food == Food {
        self.box = _AnimalBox(animal)
    }

    func eat(_ food: Food) { box.eat(food) }
    var name: String { box.name }
}
```

### Step 4: 사용

```swift
let dog = Dog()
let cat = Cat()

// ✅ 이제 같은 배열에 담을 수 있다!
let animals: [AnyAnimal<String>] = [
    AnyAnimal(dog),
    AnyAnimal(cat)
]

for animal in animals {
    animal.eat("사료")
}
// 강아지가 사료 먹는 중
// 고양이가 사료 먹는 중
```

---
<br>

## 4. 전체 흐름 정리

```
Dog (Animal, Food=String)
    ↓ _AnimalBox(dog)
    ↓ _AnimalBox<Dog> : _AnyAnimalBox<String>
    ↓                          ↑ 이 타입으로 저장 → Dog 정보 사라짐
AnyAnimal<String>.box = _AnyAnimalBox<String>
    ↓ eat("사료") 호출
    ↓ box.eat("사료")
    ↓ _AnimalBox<Dog>.eat("사료")  ← 오버라이드된 메서드 실행
    ↓ base.eat("사료")             ← 실제 Dog.eat() 호출
```

**외부에서는 AnyAnimal<String>만 보이고, 내부적으로 Dog인지 Cat인지는 알 수 없습니다**

---
<br>

## 5. 클로저 방식 — 더 간단한 구현

박스 패턴이 복잡하다면, 클로저로 캡처하는 방법도 있어요

```swift
struct AnyAnimal<Food>: Animal {
    private let _eat: (Food) -> Void
    private let _name: () -> String

    init<A: Animal>(_ animal: A) where A.Food == Food {
        // 클로저가 animal을 캡처 → 구체 타입 정보가 클로저 안에 숨겨짐
        _eat = { food in animal.eat(food) }
        _name = { animal.name }
    }

    func eat(_ food: Food) { _eat(food) }
    var name: String { _name() }
}
```

**클로저가 구체 타입을 캡처하면서 타입 정보를 숨기는 것**이 핵심입니다

박스 패턴보다 코드가 짧고, 실무에서는 이 방식을 더 많이 씁니다

---
<br>

## 6. 표준 라이브러리의 Type Erasure

Swift가 이미 제공하는 Type Erasure 타입들입니다

| Type Eraser | 원본 프로토콜 | 용도 |
|-------------|-------------|------|
| `AnySequence<T>` | Sequence | 다양한 시퀀스를 통일 |
| `AnyCollection<T>` | Collection | 다양한 컬렉션을 통일 |
| `AnyHashable` | Hashable | Dictionary 키로 사용 |
| `AnyPublisher<O, E>` | Publisher (Combine) | Publisher 체인 결과를 통일 |

```swift
// Combine에서의 실제 사용
func fetchUser() -> AnyPublisher<User, Error> {
    URLSession.shared.dataTaskPublisher(for: url)
        .map(\.data)
        .decode(type: User.self, decoder: JSONDecoder())
        .eraseToAnyPublisher()  // ← 내부 체인의 구체 타입을 지움
}
```

`eraseToAnyPublisher()`를 안 쓰면 반환 타입이 `Publishers.Decode<Publishers.Map<URLSession.DataTaskPublisher, Data>, User, JSONDecoder>` 같은 괴물이 됩니다

---
<br>

## 7. Swift 5.7+ 이후 — any 키워드로 대체 가능?

Swift 5.7부터 `any` 키워드가 도입되면서 간단한 경우는 Type Erasure 없이도 가능해졌어요

```swift
// Swift 5.7+ — any로 PAT를 타입으로 사용 가능
let animals: [any Animal] = [Dog(), Cat()]

for animal in animals {
    print(animal.name)  // ✅ 가능
    // animal.eat(???)  // ❌ Food 타입을 모르니 eat()은 호출 불가
}
```

하지만 **associatedtype을 파라미터로 사용하는 메서드는 호출할 수 없다**는 제한이 있어요

따라서 **같은 Food 타입끼리 묶어서 메서드를 호출해야 하는 경우**에는 여전히 Type Erasure가 필요합니다

| 상황 | 해결 방법 |
|------|----------|
| PAT를 그냥 담기만 할 때 | `any` 키워드로 충분 |
| 같은 associatedtype끼리 묶어서 사용 | Type Erasure 필요 |
| 반환 타입 숨기기 (Combine 등) | `eraseToAnyPublisher()` 등 Type Erasure |

---
<br>

## 면접 포인트 정리

- **Type Erasure가 왜 필요한가?**: PAT는 타입으로 직접 사용 불가 → 래퍼로 감싸서 구체 타입을 숨기고, 프로토콜 인터페이스만 노출
- **구현 방식 2가지**: 박스 패턴(상속 기반)과 클로저 캡처 패턴. 실무에서는 클로저 방식이 더 간결
- **표준 라이브러리 예시**: AnySequence, AnyPublisher 등이 모두 Type Erasure
- **any와의 관계**: Swift 5.7+ `any`로 간단한 경우는 대체 가능하지만, associatedtype을 파라미터로 쓰는 메서드 호출이 필요하면 여전히 Type Erasure 필요
- **eraseToAnyPublisher()를 왜 쓰나요?**: 내부 체인의 구체 타입이 극도로 복잡해지는 것을 방지하고, API 인터페이스를 깔끔하게 유지
