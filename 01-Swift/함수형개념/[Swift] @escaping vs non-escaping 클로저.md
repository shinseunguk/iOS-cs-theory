# Swift @escaping vs non-escaping 클로저

안녕하세요, 욱승입니다 👋

오늘은 **@escaping 클로저와 non-escaping 클로저**의 차이를 정리해보겠습니다

클로저가 함수 밖으로 "탈출"하느냐 아니냐에 따라 메모리 관리 방식이 완전히 달라져요

---
<br>

## 1. non-escaping 클로저 (기본값)

함수의 실행이 끝나기 **전에** 호출되고, 함수가 리턴되면 사라지는 클로저입니다

Swift에서 클로저 파라미터는 **기본이 non-escaping**입니다

```swift
func doSomething(closure: () -> Void) {
    closure()  // 함수 안에서 바로 실행
}   // 함수 끝 → 클로저도 사라짐
```

```
doSomething 호출 → closure() 실행 → 함수 리턴 → 끝
                   ↑ 함수 범위 안에서만 살아있음
```

### 특징

- `self` 캡처 시 **`self.` 생략 가능** (Swift 5.3+, 구조체의 경우)
- 스택에 저장 가능 → **힙 할당 불필요** → 성능상 유리
- 컴파일러가 클로저의 수명을 정확히 알 수 있어서 최적화 가능

---
<br>

## 2. @escaping 클로저

함수가 리턴된 **이후에도** 살아남는 클로저입니다

클로저가 함수 범위를 "탈출(escape)"하기 때문에 `@escaping`을 명시해야 합니다

```swift
func doSomething(closure: @escaping () -> Void) {
    DispatchQueue.main.asyncAfter(deadline: .now() + 1) {
        closure()  // 함수가 리턴된 지 1초 후에 실행
    }
}
```

```
doSomething 호출 → 함수 리턴 → ... 1초 후 ... → closure() 실행
                                                  ↑ 함수가 끝났는데도 살아있음
```

---
<br>

## 3. 탈출하는 대표적인 경우

### 비동기 작업의 콜백

```swift
func fetchData(completion: @escaping (Data) -> Void) {
    URLSession.shared.dataTask(with: url) { data, _, _ in
        if let data = data {
            completion(data)  // 네트워크 응답이 올 때 실행 → 함수는 이미 끝남
        }
    }.resume()
}
```

### 프로퍼티에 저장

```swift
class ViewModel {
    var onUpdate: (() -> Void)?

    func setHandler(handler: @escaping () -> Void) {
        self.onUpdate = handler  // 프로퍼티에 저장 → 함수 범위를 탈출
    }
}
```

### 다른 @escaping 클로저에 전달

```swift
func wrapper(closure: @escaping () -> Void) {
    DispatchQueue.global().async {
        closure()  // async 자체가 @escaping → 전달되는 closure도 @escaping이어야 함
    }
}
```

---
<br>

## 4. self 캡처 — 가장 중요한 실무 포인트

@escaping 클로저는 함수보다 오래 살아남으니, 캡처하는 값도 그만큼 유지되어야 합니다

### non-escaping — self 암묵적 캡처

```swift
struct Calculator {
    var value = 0

    func calculate(using closure: (Int) -> Int) {
        let result = closure(value)  // self. 생략 가능
        print(result)
    }
}
```

### @escaping — self 명시 필수

```swift
class ViewModel {
    var name = "욱승"

    func load(completion: @escaping () -> Void) {
        DispatchQueue.main.async {
            // ❌ 컴파일 에러 — self 명시 필요
            // print(name)

            // ✅ self 명시
            print(self.name)
        }
    }
}
```

**왜 self를 명시하게 할까?**

→ @escaping 클로저가 `self`를 강하게 캡처하면 **순환 참조(retain cycle)** 가 발생할 수 있으니, 개발자가 이를 **인지하도록** 컴파일러가 강제하는 것입니다

### 순환 참조 방지 — [weak self]

```swift
class ViewModel {
    var name = "욱승"

    func load() {
        fetchData { [weak self] data in
            guard let self else { return }
            self.name = data  // self가 해제됐으면 아무것도 안 함
        }
    }
}
```

---
<br>

## 5. 메모리 관점 비교

```
non-escaping:
┌─────────────────┐
│  함수 스택 프레임  │
│  ┌─────────────┐ │
│  │   클로저     │ │  ← 스택에 존재, 함수 끝나면 자동 해제
│  └─────────────┘ │
└─────────────────┘

@escaping:
┌─────────────────┐          ┌─────────────┐
│  함수 스택 프레임  │ ──참조──→ │   클로저 (힙)  │  ← 힙에 할당, RC로 관리
└─────────────────┘          │  캡처된 self  │
   함수 끝나도 ↑ 살아있음       └─────────────┘
```

| 구분 | non-escaping | @escaping |
|------|-------------|-----------|
| 저장 위치 | 스택 (최적화 가능) | 힙 |
| 수명 | 함수 범위 내 | 함수 리턴 이후에도 유지 |
| self 캡처 | 암묵적 (명시 불필요) | 명시적 (`self.` 필수) |
| 순환 참조 위험 | 없음 | 있음 (`[weak self]` 필요) |
| 성능 | 더 빠름 | 힙 할당 비용 있음 |

---
<br>

## 6. @escaping이 필요 없는데 붙이면?

동작은 하지만 **불필요한 힙 할당**이 발생하고, 컴파일러 최적화를 방해합니다

```swift
// ❌ 불필요한 @escaping — 바로 실행하는데 escaping을 붙임
func process(closure: @escaping () -> Void) {
    closure()
}

// ✅ non-escaping으로 충분
func process(closure: () -> Void) {
    closure()
}
```

Swift가 기본을 non-escaping으로 정한 이유가 바로 이것 — **성능과 안전성 모두 non-escaping이 유리**하기 때문입니다

---
<br>

## 7. @autoclosure와 함께 쓰기

`@autoclosure`와 `@escaping`은 함께 사용할 수 있습니다

```swift
func assert(_ condition: @autoclosure @escaping () -> Bool,
            _ message: String) {
    DispatchQueue.global().async {
        if !condition() {
            print("Assertion failed: \(message)")
        }
    }
}

assert(1 + 1 == 2, "수학이 깨졌다")
// 1 + 1 == 2 표현식이 자동으로 클로저로 감싸지고, 비동기로 실행됨
```

---
<br>

## 면접 포인트 정리

- **non-escaping이 기본인 이유**: 스택 할당으로 성능이 좋고, 순환 참조 위험이 없으며, 컴파일러 최적화가 가능하기 때문
- **@escaping은 언제 필요한가**: 비동기 콜백, 프로퍼티 저장, 다른 @escaping 클로저에 전달 등 함수 범위를 벗어나야 할 때
- **self 명시를 강제하는 이유**: @escaping 클로저의 강한 캡처로 인한 순환 참조를 개발자가 인지하도록
- **[weak self] vs [unowned self]**: self가 먼저 해제될 수 있으면 `weak`, 클로저와 수명이 같다고 확신하면 `unowned` (확신 없으면 `weak` 사용)
- **메모리 차이**: non-escaping은 스택, @escaping은 힙. 불필요한 @escaping은 성능 저하를 유발
