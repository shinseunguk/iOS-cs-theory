# RxSwift 기초 문법 정리

안녕하세요, 욱승입니다 👋

오늘은 **RxSwift의 기초 문법**을 정리해봤습니다

비동기 처리를 스트림 기반으로 깔끔하게 다룰 수 있게 해주는 라이브러리인데요

면접에서도 "RxSwift 써보셨나요?"로 시작해서 "Subject 종류 차이가 뭔가요?"까지 꼬리질문 오는 단골 주제입니다

---
<br>

## Observable이 뭐냐면

**데이터를 방출하는 스트림**이에요

Observable은 3가지 이벤트를 발생시킵니다

| 이벤트 | 설명 |
|--------|------|
| `onNext` | 새로운 값을 방출 |
| `onError` | 에러 발생 후 스트림 종료 |
| `onCompleted` | 정상 종료 |

```
시간 →  ──①──②──③──|
            next next next completed
```

Observable은 구독(subscribe)하기 전까지는 아무것도 안 해요

**Cold Observable**이라고도 하는데, 구독해야 비로소 데이터를 방출하기 시작합니다

---
<br>

## Observable 생성 방법

```swift
// just - 단일 값 하나만 방출
let observable1 = Observable.just(1)           // 1 → completed

// of - 여러 값을 순서대로 방출
let observable2 = Observable.of(1, 2, 3)       // 1, 2, 3 → completed

// from - 배열을 개별 요소로 방출
let observable3 = Observable.from([1, 2, 3])   // 1, 2, 3 → completed

// of vs from 차이 주의!
Observable.of([1, 2, 3])    // [1, 2, 3] 배열 자체를 하나로 방출
Observable.from([1, 2, 3])  // 1, 2, 3 개별로 방출
```

직접 생성도 가능합니다

```swift
let observable4 = Observable<String>.create { observer in
    observer.onNext("Hello")
    observer.onNext("RxSwift")
    observer.onCompleted()
    return Disposables.create()  // 구독 해제 시 정리 작업
}
```

---
<br>

## Subscribe (구독)

Observable은 **구독해야 동작**합니다

```swift
let observable = Observable.of(1, 2, 3)

observable.subscribe(
    onNext: { value in
        print("값: \(value)")
    },
    onError: { error in
        print("에러: \(error)")
    },
    onCompleted: {
        print("완료!")
    },
    onDisposed: {
        print("해제됨")
    }
)

// 출력:
// 값: 1
// 값: 2
// 값: 3
// 완료!
// 해제됨
```

간단하게 onNext만 받을 수도 있어요

```swift
observable
    .subscribe(onNext: { print($0) })
```

---
<br>

## DisposeBag (메모리 관리)

구독을 해제하지 않으면 **메모리 누수**가 발생해요

DisposeBag은 자신이 deinit될 때 담겨있는 구독을 전부 해제해줍니다

```swift
class MyViewController: UIViewController {
    let disposeBag = DisposeBag()  // VC가 해제되면 구독도 같이 해제

    override func viewDidLoad() {
        super.viewDidLoad()

        Observable.of(1, 2, 3)
            .subscribe(onNext: { print($0) })
            .disposed(by: disposeBag)  // disposeBag에 담기
    }
}
```

```
ViewController deinit → DisposeBag deinit → 모든 Disposable dispose()
```

수동으로 해제하고 싶으면 이렇게도 가능합니다

```swift
let disposable = observable.subscribe(onNext: { print($0) })
disposable.dispose()  // 직접 해제
```

---
<br>

## Subject (Observable + Observer)

Observable은 값을 방출만 하고, Observer는 값을 받기만 하는데

**Subject는 둘 다 가능**해요. 값을 직접 넣을 수도 있고, 구독할 수도 있습니다

### PublishSubject

구독 **이후**의 이벤트만 받습니다

```swift
let subject = PublishSubject<String>()

subject.onNext("A")  // 구독 전이라 무시됨

subject.subscribe(onNext: { print($0) })
    .disposed(by: disposeBag)

subject.onNext("B")  // 출력: B
subject.onNext("C")  // 출력: C
```

```
구독 시점
   ↓
───A───B───C───
       ↑   ↑
      받음 받음  (A는 못 받음)
```

### BehaviorSubject

**초기값**이 필요하고, 구독 시 **최신 값 1개**를 즉시 받습니다

```swift
let subject = BehaviorSubject(value: "초기값")

subject.onNext("A")

subject.subscribe(onNext: { print($0) })
    .disposed(by: disposeBag)
// 출력: A (구독 시점의 최신 값)

subject.onNext("B")  // 출력: B
```

```
초기값──A──구독──B──
          ↓    ↓
         A받음 B받음  (최신값 A를 즉시 전달)
```

### ReplaySubject

**버퍼 크기만큼** 과거 이벤트를 재방출해요

```swift
let subject = ReplaySubject<String>.create(bufferSize: 2)

subject.onNext("A")
subject.onNext("B")
subject.onNext("C")

subject.subscribe(onNext: { print($0) })
    .disposed(by: disposeBag)
// 출력: B, C (최근 2개)
```

### Subject 비교 정리

| Subject | 초기값 | 구독 시 받는 값 | 사용 예시 |
|---------|--------|----------------|----------|
| PublishSubject | X | 구독 이후만 | 버튼 탭 이벤트 |
| BehaviorSubject | O | 최신 1개 + 이후 | 현재 상태 값 (로딩 상태 등) |
| ReplaySubject | X | 버퍼만큼 + 이후 | 최근 채팅 메시지 |

---
<br>

## 주요 Operator

### map - 값 변환

```swift
Observable.of(1, 2, 3)
    .map { $0 * 10 }
    .subscribe(onNext: { print($0) })
    .disposed(by: disposeBag)
// 출력: 10, 20, 30
```

### filter - 조건 필터링

```swift
Observable.of(1, 2, 3, 4, 5)
    .filter { $0 > 3 }
    .subscribe(onNext: { print($0) })
    .disposed(by: disposeBag)
// 출력: 4, 5
```

### flatMap - Observable로 변환 후 평탄화

내부에서 또 다른 Observable을 만들어야 할 때 사용해요

```swift
struct User {
    let name: BehaviorSubject<String>
}

let user = User(name: BehaviorSubject(value: "욱승"))

Observable.just(user)
    .flatMap { $0.name }  // User → name Observable로 전환
    .subscribe(onNext: { print($0) })
    .disposed(by: disposeBag)
// 출력: 욱승

user.name.onNext("변경됨")
// 출력: 변경됨  (내부 Observable 변화도 추적)
```

### distinctUntilChanged - 연속 중복 제거

```swift
Observable.of(1, 1, 2, 2, 3, 1)
    .distinctUntilChanged()
    .subscribe(onNext: { print($0) })
    .disposed(by: disposeBag)
// 출력: 1, 2, 3, 1  (연속된 중복만 제거)
```

### combineLatest - 여러 Observable 조합

```swift
let id = PublishSubject<String>()
let pw = PublishSubject<String>()

Observable.combineLatest(id, pw) { id, pw in
    return !id.isEmpty && !pw.isEmpty
}
.subscribe(onNext: { isValid in
    print("로그인 버튼 활성화: \(isValid)")
})
.disposed(by: disposeBag)

id.onNext("user")    // 아직 pw가 없어서 이벤트 X
pw.onNext("1234")    // 출력: 로그인 버튼 활성화: true
pw.onNext("")        // 출력: 로그인 버튼 활성화: false
```

### merge - 여러 Observable 합치기

```swift
let a = PublishSubject<String>()
let b = PublishSubject<String>()

Observable.merge(a, b)
    .subscribe(onNext: { print($0) })
    .disposed(by: disposeBag)

a.onNext("A1")  // 출력: A1
b.onNext("B1")  // 출력: B1
a.onNext("A2")  // 출력: A2
```

### withLatestFrom - 트리거 기반 조합

```swift
let button = PublishSubject<Void>()     // 버튼 탭
let textField = PublishSubject<String>() // 텍스트 입력

button.withLatestFrom(textField)
    .subscribe(onNext: { print("검색어: \($0)") })
    .disposed(by: disposeBag)

textField.onNext("Rx")
textField.onNext("RxSwift")
button.onNext(())       // 출력: 검색어: RxSwift
textField.onNext("Swift")
button.onNext(())       // 출력: 검색어: Swift
```

---
<br>

## Scheduler (스레드 관리)

```swift
observable
    .observe(on: ConcurrentDispatchQueueScheduler(qos: .background))  // 이후 작업을 백그라운드에서
    .map { /* 백그라운드에서 데이터 가공 */ }
    .observe(on: MainScheduler.instance)  // 이후 작업을 메인에서
    .subscribe(onNext: { /* UI 업데이트 */ })
    .disposed(by: disposeBag)
```

| Scheduler | 설명 |
|-----------|------|
| `MainScheduler.instance` | 메인 스레드 (UI 작업) |
| `ConcurrentDispatchQueueScheduler` | 백그라운드 (네트워크, 연산) |
| `SerialDispatchQueueScheduler` | 시리얼 큐 |

---
<br>

## 실전 예시: 검색 기능

```swift
searchBar.rx.text.orEmpty       // 텍스트 변경 이벤트
    .debounce(.milliseconds(300), scheduler: MainScheduler.instance)  // 0.3초 대기
    .distinctUntilChanged()      // 같은 텍스트 중복 방지
    .filter { !$0.isEmpty }      // 빈 문자열 무시
    .flatMapLatest { query in    // 최신 요청만 유지
        APIService.search(query)
    }
    .observe(on: MainScheduler.instance)
    .subscribe(onNext: { results in
        self.updateUI(results)
    })
    .disposed(by: disposeBag)
```

이게 RxSwift의 강력한 점이에요

debounce, distinctUntilChanged, flatMapLatest를 체이닝해서 불필요한 API 호출을 줄이고 최신 결과만 받을 수 있습니다

---
<br>

## 면접 포인트 정리

- **Observable vs Subject 차이**: Observable은 읽기 전용, Subject는 읽기+쓰기
- **Hot vs Cold Observable**: Cold는 구독해야 방출, Hot(Subject)은 구독 여부와 관계없이 방출
- **DisposeBag이 필요한 이유**: 구독 해제를 안 하면 메모리 누수 발생
- **flatMap vs flatMapLatest**: flatMap은 모든 내부 Observable 유지, flatMapLatest는 최신 것만 유지
- **Scheduler**: observeOn은 이후 체인의 스레드, subscribeOn은 구독 시작 스레드를 결정
