# Swift ARC 동작 방식 정리

안녕하세요, 욱승입니다 👋

이전 글에서 Class는 Heap에 할당되고 refCount로 관리된다고 했는데요

오늘은 그 **refCount를 관리하는 주체인 ARC(Automatic Reference Counting)** 가 어떻게 동작하는지 깊게 파봤습니다.

면접에서도 "ARC가 뭔가요?"로 시작해서 "순환 참조 어떻게 해결하나요?"까지 꼬리질문 오는 단골 주제입니다

---
<br>

## ARC가 뭐냐면

**Automatic Reference Counting**, 말 그대로 참조 횟수를 자동으로 세주는 메모리 관리 방식이에요

class 인스턴스가 생길 때마다 ARC가 해당 객체를 몇 군데서 참조하고 있는지 추적하고
참조하는 곳이 0개가 되면 메모리에서 해제해요

```
인스턴스 생성 → refCount = 1
변수가 참조 추가 → refCount += 1
변수가 참조 해제 → refCount -= 1
refCount == 0 → 메모리 해제 (deinit 호출)
```

GC(Garbage Collection)와 다른 점은, GC는 런타임에 주기적으로 쓸어가는 방식이고
ARC는 **컴파일 타임에 retain/release 코드를 삽입**해서 동작한다는 거에요

그래서 GC처럼 예측 불가능한 성능 저하가 없습니다

---
<br>

## 기본 동작 흐름

```swift
class Person {
    let name: String
    init(name: String) {
        self.name = name
        print("\(name) 생성")
    }
    deinit {
        print("\(name) 해제")
    }
}

var ref1: Person? = Person(name: "욱승")  // refCount = 1
var ref2 = ref1                           // refCount = 2
var ref3 = ref1                           // refCount = 3

ref1 = nil                                // refCount = 2
ref2 = nil                                // refCount = 1
ref3 = nil                                // refCount = 0 → deinit 호출!
```

### 메모리에서 보면

```
[ref3 = nil 시점]

Stack                    Heap
┌──────────────┐        ┌─────────────────┐
│ ref1: nil    │        │ Person          │
│ ref2: nil    │        │  name: "욱승"   │
│ ref3: nil    │        │  refCount: 0    │ ← 해제 대상!
└──────────────┘        └─────────────────┘
```

모든 참조가 nil이 되면 refCount가 0이 되고, 그 즉시 deinit이 호출되면서 Heap 메모리가 반환됩니다

---
<br>

## 순환 참조 - ARC의 함정

ARC가 자동이라고 만능은 아닙니다

두 객체가 서로를 참조하면 refCount가 영원히 0이 안 되는 상황이 생겨요
이걸 **강한 순환 참조(Strong Reference Cycle)** 라고 합니다

```swift
class Person {
    let name: String
    var pet: Dog?
    init(name: String) { self.name = name }
    deinit { print("\(name) 해제") }
}

class Dog {
    let name: String
    var owner: Person?
    init(name: String) { self.name = name }
    deinit { print("\(name) 해제") }
}

var person: Person? = Person(name: "욱승")   // Person refCount = 1
var dog: Dog? = Dog(name: "초코")            // Dog refCount = 1

person?.pet = dog       // Dog refCount = 2
dog?.owner = person     // Person refCount = 2

person = nil            // Person refCount = 1 (Dog가 아직 참조 중)
dog = nil               // Dog refCount = 1 (Person이 아직 참조 중)

// 둘 다 deinit 안 불림!! → 메모리 누수 💀
```

### 그림으로 보면

```
person = nil, dog = nil 한 뒤에도:

Stack                    Heap
┌──────────────┐        ┌──────────────────┐
│ person: nil  │        │ Person           │
│ dog: nil     │        │  pet ──────────┐ │
└──────────────┘        │  refCount: 1   │ │
                        └────────────────┼─┘
                              ▲          │
                              │          ▼
                        ┌─────┼──────────────┐
                        │     │  Dog         │
                        │  owner             │
                        │  refCount: 1       │
                        └────────────────────┘

→ Stack에서 접근할 방법은 없는데 Heap에서 서로 붙잡고 있음
→ 메모리 누수!
```

Stack에서 접근할 방법은 사라졌는데 Heap에서 서로를 붙잡고 있으니까 영원히 해제가 안 됩니다
이게 ARC 쓸 때 가장 조심해야 하는 부분이에요

---
<br>

## 해결법 1: weak (약한 참조)

`weak`은 참조는 하되 **refCount를 올리지 않는** 참조에요
참조하던 객체가 해제되면 자동으로 nil이 됩니다

그래서 반드시 `Optional` 타입이어야 하고, `var`로 선언해야 합니다

```swift
class Dog {
    let name: String
    weak var owner: Person?  // weak으로 변경!
    init(name: String) { self.name = name }
    deinit { print("\(name) 해제") }
}

var person: Person? = Person(name: "욱승")   // Person refCount = 1
var dog: Dog? = Dog(name: "초코")            // Dog refCount = 1

person?.pet = dog       // Dog refCount = 2
dog?.owner = person     // Person refCount = 1 (weak이라 안 올라감!)

person = nil            // Person refCount = 0 → 해제! → pet 참조 해제 → Dog refCount = 1
dog = nil               // Dog refCount = 0 → 해제!

// "욱승 해제"
// "초코 해제"  ← 정상적으로 둘 다 해제됨! ✅
```

### 메모리 흐름

```
weak 적용 후:

Person ──(strong)──► Dog
Person ◄──(weak)──── Dog     ← refCount 안 올림

person = nil 시점:
  Person refCount 0 → 해제
  → pet이 들고 있던 Dog 참조 해제 → Dog refCount 1
dog = nil 시점:
  Dog refCount 0 → 해제
```

---
<br>

## 해결법 2: unowned (미소유 참조)

`unowned`도 refCount를 올리지 않지만, **nil이 되지 않습니다**

참조 대상이 먼저 해제되면 dangling pointer가 되어서 접근 시 **크래시**가 납니다 💥

```swift
class Customer {
    let name: String
    var card: CreditCard?
    init(name: String) { self.name = name }
    deinit { print("\(name) 해제") }
}

class CreditCard {
    let number: Int
    unowned let owner: Customer  // 카드는 반드시 주인이 있음
    init(number: Int, owner: Customer) {
        self.number = number
        self.owner = owner
    }
    deinit { print("카드 \(number) 해제") }
}

var customer: Customer? = Customer(name: "욱승")
customer?.card = CreditCard(number: 1234, owner: customer!)

customer = nil
// "욱승 해제"
// "카드 1234 해제"  ← 둘 다 정상 해제 ✅
```

`unowned`은 **"이 참조 대상은 나보다 먼저 해제되지 않는다"** 라고 확신할 수 있을 때 쓰는 거에요

---
<br>

## 클로저에서의 순환 참조

실무에서 순환 참조 가장 많이 만나는 곳이 바로 **클로저**입니다

클로저는 주변 변수를 캡처하는데, class 인스턴스의 프로퍼티로 클로저를 저장하면
클로저가 self를 캡처 → self가 클로저를 소유 → 순환 참조 발생!

```swift
class ViewModel {
    var name = "욱승"
    var onUpdate: (() -> Void)?

    func setup() {
        onUpdate = {
            print(self.name)  // self를 강하게 캡처! ⚠️
        }
    }

    deinit { print("ViewModel 해제") }
}

var vm: ViewModel? = ViewModel()
vm?.setup()
vm = nil  // deinit 안 불림!! 💀
```

### 캡처 리스트로 해결

```swift
func setup() {
    onUpdate = { [weak self] in
        guard let self else { return }
        print(self.name)
    }
}
```

`[weak self]`로 캡처하면 클로저가 self의 refCount를 올리지 않아서 순환이 깨집니다

---
<br>

## weak vs unowned 언제 쓰냐면

| | weak | unowned |
|:--|:--:|:--:|
| refCount 증가 | ❌ | ❌ |
| Optional 여부 | Optional (nil 가능) | Non-Optional |
| 대상 해제 시 | 자동으로 nil | 크래시 💥 |
| 사용 시점 | 대상이 먼저 해제될 수 있을 때 | 대상이 나보다 먼저 해제 안 될 때 |
| 대표적인 예 | delegate, 클로저 캡처 | 부모-자식 관계 (자식 → 부모) |

실무에서는 확신이 없으면 **일단 weak 쓰는 게 안전**합니다
unowned은 생명주기가 명확할 때만 쓰는 걸 추천해요

---
<br>

## ARC 동작 타이밍 정리

```
컴파일 타임:
  Swift 컴파일러가 코드를 분석해서
  적절한 위치에 retain(참조 +1) / release(참조 -1) 코드를 자동 삽입

런타임:
  인스턴스 생성 → retain → refCount = 1
  변수에 할당  → retain → refCount += 1
  스코프 종료  → release → refCount -= 1
  refCount == 0 → deinit 호출 → 메모리 해제
```

컴파일 타임에 삽입되기 때문에 런타임 오버헤드가 GC보다 적고
메모리 해제 시점이 예측 가능하다는 게 ARC의 큰 장점이에요

---
<br>

## 한눈에 정리

```
                    ARC 핵심 구조

 ┌─────────────────────────────────────────┐
 │         Strong Reference (기본)          │
 │         → refCount 올림                 │
 │         → 순환 참조 위험 있음            │
 ├─────────────────────────────────────────┤
 │         Weak Reference                  │
 │         → refCount 안 올림              │
 │         → 대상 해제 시 nil              │
 │         → Optional, var 필수            │
 ├─────────────────────────────────────────┤
 │         Unowned Reference               │
 │         → refCount 안 올림              │
 │         → 대상 해제 시 크래시           │
 │         → Non-Optional 가능             │
 └─────────────────────────────────────────┘
```

---
<br>

## 결론

ARC 덕분에 수동으로 retain/release 안 해도 되지만, **순환 참조는 개발자가 직접 신경 써야** 합니다

특히 클로저에서 `[weak self]` 빼먹는 거 진짜 자주 일어나는 실수에요
Instruments의 Leaks나 Memory Graph Debugger로 주기적으로 확인하는 습관을 들이면 좋습니다

정리하면:
- ARC는 컴파일 타임에 retain/release를 삽입하는 방식
- 순환 참조가 생기면 메모리 누수 발생
- `weak`, `unowned`로 순환을 끊어줘야 함
- 클로저 캡처 시 `[weak self]`는 거의 필수

이전 글에서 다뤘던 Struct vs Class 메모리 구조와 같이 보면 더 이해가 잘 될 거에요 🫡

문의는 댓글!

---

Ref.
- [Apple Developer - Automatic Reference Counting](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/automaticreferencecounting/)
- [Apple Developer - Choosing Between Structures and Classes](https://developer.apple.com/documentation/swift/choosing-between-structures-and-classes)