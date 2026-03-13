# Swift map vs flatMap 차이

안녕하세요, 욱승입니다 👋

오늘은 Swift에서 자주 쓰이는 **map**과 **flatMap**의 차이를 정리해보겠습니다

둘 다 값을 변환하는 고차함수지만, 중첩된 구조를 어떻게 처리하느냐에서 결정적 차이가 있어요

---
<br>

## 1. map — 변환만 한다

`map`은 컨테이너 안의 각 요소에 변환 함수를 적용하고, **결과를 그대로 다시 감싸서** 반환합니다

```swift
let numbers = [1, 2, 3]

let doubled = numbers.map { $0 * 2 }
// [2, 4, 6]
```

핵심: **컨테이너 구조는 그대로 유지**하고, 안의 값만 바꿉니다

---
<br>

## 2. flatMap — 변환 + 평탄화

`flatMap`은 변환 결과가 **중첩 컨테이너**일 때, 한 겹을 벗겨서 평탄화(flatten)합니다

### Array에서의 flatMap

```swift
let sentences = ["Hello World", "Swift is great"]

// map → 2차원 배열
let mapped = sentences.map { $0.split(separator: " ") }
// [["Hello", "World"], ["Swift", "is", "great"]]

// flatMap → 1차원 배열로 평탄화
let flatMapped = sentences.flatMap { $0.split(separator: " ") }
// ["Hello", "World", "Swift", "is", "great"]
```

### Optional에서의 flatMap

```swift
let input: String? = "42"

// map → Optional<Optional<Int>> (이중 옵셔널)
let mapped = input.map { Int($0) }
// Optional(Optional(42)) → Int??

// flatMap → Optional<Int> (한 겹 벗겨짐)
let flatMapped = input.flatMap { Int($0) }
// Optional(42) → Int?
```

---
<br>

## 3. 핵심 차이 한눈에 보기

```
map:     [A] → (A → B)  → [B]        // 변환만
flatMap: [A] → (A → [B]) → [B]       // 변환 + 평탄화

map:     A?  → (A → B)  → B?         // 변환만
flatMap: A?  → (A → B?) → B?         // 변환 + 평탄화
```

| 구분 | map | flatMap |
|------|-----|---------|
| 변환 함수 반환값 | 단일 값 | 컨테이너(배열, 옵셔널 등) |
| 결과 구조 | 중첩될 수 있음 | 한 겹 벗겨짐 |
| Array 예시 | `[[1,2],[3,4]]` | `[1,2,3,4]` |
| Optional 예시 | `Int??` | `Int?` |

---
<br>

## 4. compactMap — nil 제거 전용

Swift 4.1부터 **Optional을 반환하는 Array의 flatMap**은 `compactMap`으로 분리되었습니다

```swift
let strings = ["1", "two", "3", "four"]

// ❌ flatMap으로 nil 필터링 → deprecated 경고
let numbers = strings.flatMap { Int($0) }

// ✅ compactMap 사용
let numbers = strings.compactMap { Int($0) }
// [1, 3]
```

**분리된 이유**: flatMap이 "평탄화"와 "nil 제거" 두 가지 역할을 하니 의미가 모호해서, 역할을 명확히 나눈 것입니다

| 함수 | 역할 |
|------|------|
| `map` | 변환만 |
| `flatMap` | 변환 + 중첩 컨테이너 평탄화 |
| `compactMap` | 변환 + nil 제거 |

---
<br>

## 5. 실전 활용 예시

### 중첩 배열 처리

```swift
struct User {
    let name: String
    let hobbies: [String]
}

let users = [
    User(name: "욱승", hobbies: ["독서", "코딩"]),
    User(name: "철수", hobbies: ["게임", "운동", "코딩"])
]

// 모든 취미를 하나의 배열로
let allHobbies = users.flatMap { $0.hobbies }
// ["독서", "코딩", "게임", "운동", "코딩"]
```

### 옵셔널 체이닝 대체

```swift
let dict: [String: [String: Int]] = [
    "user": ["age": 25]
]

// flatMap으로 안전하게 중첩 접근
let age = dict["user"].flatMap { $0["age"] }
// Optional(25)

// 없는 키 접근
let missing = dict["admin"].flatMap { $0["age"] }
// nil
```

### JSON 파싱에서

```swift
struct Response {
    let data: Data?
}

let response = Response(data: someData)

// map → UIImage??
let image1 = response.data.map { UIImage(data: $0) }

// flatMap → UIImage?
let image2 = response.data.flatMap { UIImage(data: $0) }
```

---
<br>

## 6. 동작 원리 — 시그니처로 이해하기

```swift
// Array
func map<T>(_ transform: (Element) -> T) -> [T]
func flatMap<T>(_ transform: (Element) -> [T]) -> [T]
func compactMap<T>(_ transform: (Element) -> T?) -> [T]

// Optional
func map<T>(_ transform: (Wrapped) -> T) -> T?
func flatMap<T>(_ transform: (Wrapped) -> T?) -> T?
```

flatMap의 시그니처를 보면:
- **Array**: 변환 함수가 `[T]`를 반환 → 결과를 합쳐서 `[T]`로 (이중 배열 → 단일 배열)
- **Optional**: 변환 함수가 `T?`를 반환 → 결과를 합쳐서 `T?`로 (이중 옵셔널 → 단일 옵셔널)

**`flat`의 의미는 동일합니다 — 중첩된 한 겹을 벗긴다**

---
<br>

## 면접 포인트 정리

- **map vs flatMap 차이**: map은 변환만, flatMap은 변환 + 평탄화. 변환 결과가 컨테이너일 때 중첩을 한 겹 벗겨줌
- **compactMap은 왜 분리됐나?**: flatMap이 "평탄화"와 "nil 제거"를 동시에 담당해서 의미가 모호했기 때문에 Swift 4.1에서 분리
- **Optional에서 flatMap을 왜 쓰나?**: 변환 결과가 Optional일 때 이중 옵셔널(`T??`)을 방지하기 위해
- **시그니처의 핵심 차이**: map의 변환 함수는 `(A) → B`, flatMap의 변환 함수는 `(A) → Container<B>`
