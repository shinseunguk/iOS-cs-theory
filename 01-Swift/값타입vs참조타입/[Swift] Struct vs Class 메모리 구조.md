# Swift Struct vs Class, 메모리 구조 정리

안녕하세요, 욱승입니다 👋

오늘은 면접 단골 질문이자 실제로 코드 짜다 보면 한 번씩 고민되는 그것..

**"이거 struct로 해야 하나 class로 해야 하나"**

에 대해서 메모리 레벨로 정리해봤습니다.

그냥 "struct는 값 타입, class는 참조 타입" 이것만 외우면 언젠가 한 번은 당하거든요

---
<br>

## Stack이냐 Heap이냐, 그게 핵심

```
┌─────────────────────────────────────────────┐
│                  Memory                      │
│                                             │
│  ┌──────────────┐    ┌──────────────────┐   │
│  │    Stack     │    │      Heap        │   │
│  │              │    │                  │   │
│  │  struct 값   │    │  class 인스턴스  │   │
│  │  (직접 저장) │    │  (참조로 접근)   │   │
│  │              │    │                  │   │
│  │ [a: 1, b: 2] │    │ [ref] ────────► │   │
│  └──────────────┘    └──────────────────┘   │
└─────────────────────────────────────────────┘
```

struct는 Stack에 값 자체를 저장하고
class는 Heap에 인스턴스를 만들고 Stack엔 그 주소(포인터)만 들고 있어요

이 차이 하나가 복사 동작, ARC, 성능까지 전부 다르게 만듭니당

---
<br>

## Struct - 복사하면 진짜로 복사

```swift
struct Point {
    var x: Int
    var y: Int
}

var a = Point(x: 1, y: 2)
var b = a        // 복사 발생

b.x = 99

print(a.x)  // 1  ← 영향 없음
print(b.x)  // 99
```

`b = a` 할 때 값 전체가 복사되기 때문에 `b`를 아무리 수정해도 `a`는 멀쩡합니당

### 메모리에서 어떻게 생겼냐면

```
Stack
┌────────────────┐
│  a: [x:1, y:2] │  ← 값 자체
│  b: [x:99,y:2] │  ← 완전히 독립된 복사본
└────────────────┘
```

- Stack에 직접 저장되니까 접근 속도가 빠름
- 스코프 끝나면 알아서 해제, ARC 필요 없어요
- 스레드 간에 공유해도 비교적 안전함당

---

## Class - 복사하면 주소만 복사됨다

```swift
class Point {
    var x: Int
    var y: Int
    init(x: Int, y: Int) { self.x = x; self.y = y }
}

var a = Point(x: 1, y: 2)
var b = a        // 참조 복사 (같은 객체 가리킴)

b.x = 99

print(a.x)  // 99  ← 같이 바뀜!!
print(b.x)  // 99
```

`b = a` 해도 새 객체가 생기는 게 아니라 같은 Heap 주소를 두 변수가 같이 가리키게 됩니당
그래서 `b` 바꾸면 `a`도 같이 바뀌는 거에요 처음에 이거 몰라서 당하는 분들 꽤 있더라구요

### 메모리에서 보면

```
Stack                    Heap
┌──────────────┐        ┌─────────────────┐
│ a: [0x1A2B] ─┼──────► │ Point           │
│ b: [0x1A2B] ─┼──────► │  x: 99, y: 2   │
└──────────────┘        │  refCount: 2    │
                        └─────────────────┘
```

- Heap 할당 비용 + ARC가 refCount를 계속 추적하는 비용이 있음다
- refCount가 0이 되는 순간 메모리 해제

---

## Copy-on-Write, Array는 왜 느리지 않을까

Array, Dictionary 같은 컬렉션은 struct인데 데이터는 Heap에 저장함당
그럼 매번 복사할 때마다 Heap 접근이 생기는 거 아니냐? 싶지만

여기서 CoW(Copy-on-Write)가 등장함다

```swift
var arr1 = [1, 2, 3]   // Heap에 버퍼 생성
var arr2 = arr1         // 아직 실제 복사 안 함 (버퍼 공유 중)

arr2.append(4)          // 이 시점에서야 실제 복사 발생!
```

```
수정 전:             수정 후:
arr1 ──┐            arr1 ──► [1,2,3]
       ├──► [1,2,3]
arr2 ──┘            arr2 ──► [1,2,3,4]  ← 새 버퍼
```

수정하기 전까지는 같은 버퍼 공유하다가 실제로 값이 달라지는 순간에만 복사함당
덕분에 불필요한 복사 비용을 줄일 수 있어요

---

## Struct 안에 Class 있으면 함정 있음다 (중요)

```swift
class Engine { var hp: Int = 100 }

struct Car {
    var name: String     // Value Type
    var engine: Engine   // Reference Type
}

var car1 = Car(name: "A", engine: Engine())
var car2 = car1

car2.name = "B"           // car1.name은 "A" 그대로 ✅
car2.engine.hp = 999      // ⚠️ car1.engine.hp도 999로 바뀜!!
```

struct 복사해도 내부에 class 프로퍼티 있으면 그건 참조를 공유함당
이걸 **얕은 복사(Shallow Copy)** 라고 하는데 이게 은근 예상치 못한 버그의 주범이 됨..

struct 쓴다고 무조건 안심하면 안 된다는 뜻이에요

---

## 한눈에 비교

| | Struct | Class |
|:--|:--:|:--:|
| 저장 위치 | Stack | Heap |
| 복사 방식 | 값 복사 | 참조 복사 |
| ARC | 불필요 | 필요 |
| 할당 속도 | 빠름 | 상대적으로 느림 |
| 상속 | ❌ | ✅ |
| 멀티스레드 안전성 | 비교적 안전 | 주의 필요 |

---

## 그래서 뭘 써야 하냐면

**Struct 쓰는 경우**

독립적인 데이터 표현, 값이 공유될 필요 없을 때

```swift
struct UserProfile { }
struct Coordinate { }
struct APIResponse { }
```

**Class 쓰는 경우**

상태를 공유해야 하거나, 상속이 필요하거나, UIKit처럼 프레임워크가 class를 요구할 때

```swift
class NetworkManager { }   // 싱글톤
class ViewController { }   // UIKit
class DatabaseConnection { }
```

---

## 결론

Apple도 공식적으로 **기본은 Struct, class가 꼭 필요한 상황에서만 Class** 쓰라고 권장하고 있음당

처음엔 그냥 값 타입/참조 타입으로 외웠다가 메모리 구조를 이해하고 나면 왜 그런 권장이 나왔는지 납득이 됨다
Heap 할당이랑 ARC 오버헤드를 피할 수 있으면 피하는 게 성능 면에서 이득이니까요

정리하고 나니까 뭔가 개운하네요 🥺

문의는 댓글!

---

Ref.
- [Apple Developer - Choosing Between Structures and Classes](https://developer.apple.com/documentation/swift/choosing-between-structures-and-classes)