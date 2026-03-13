# RxSwift 심화 과정 정리

안녕하세요, 욱승입니다 👋

오늘은 **RxSwift 심화 과정**을 정리해봤습니다

기초 문법을 넘어서, 실무에서 자주 마주치는 패턴들과 면접에서 깊이 있는 답변을 할 수 있는 주제들을 다뤄볼게요

---
<br>

## Traits (Single, Completable, Maybe)

Observable은 범용적이지만, 때로는 **더 명확한 의미**를 가진 타입이 필요해요

Traits는 Observable을 래핑해서 특정 상황에 맞는 제약을 걸어둔 타입입니다

### Single

**성공(값 1개) 또는 실패** 두 가지만 있어요. API 호출에 딱 맞습니다

```swift
func fetchUser(id: Int) -> Single<User> {
    return Single.create { single in
        APIService.getUser(id: id) { result in
            switch result {
            case .success(let user):
                single(.success(user))   // 값 하나 방출 후 종료
            case .failure(let error):
                single(.failure(error))   // 에러 후 종료
            }
        }
        return Disposables.create()
    }
}

fetchUser(id: 1)
    .subscribe(
        onSuccess: { user in print(user.name) },
        onFailure: { error in print(error) }
    )
    .disposed(by: disposeBag)
```

### Completable

**값 없이 완료 또는 실패**만 있어요. 저장, 삭제 같은 작업에 적합합니다

```swift
func saveToCache(data: Data) -> Completable {
    return Completable.create { completable in
        do {
            try CacheManager.save(data)
            completable(.completed)
        } catch {
            completable(.error(error))
        }
        return Disposables.create()
    }
}

saveToCache(data: someData)
    .subscribe(
        onCompleted: { print("저장 완료") },
        onError: { error in print("저장 실패: \(error)") }
    )
    .disposed(by: disposeBag)
```

### Maybe

**값 하나 방출, 빈 완료, 또는 실패** 세 가지가 가능해요. 캐시 조회처럼 값이 있을 수도 없을 수도 있는 경우에 사용합니다

```swift
func loadFromCache(key: String) -> Maybe<Data> {
    return Maybe.create { maybe in
        if let data = CacheManager.get(key) {
            maybe(.success(data))    // 캐시 히트
        } else {
            maybe(.completed)        // 캐시 미스 (에러는 아님)
        }
        return Disposables.create()
    }
}
```

### Traits 비교 정리

| Trait | 방출 이벤트 | 사용 예시 |
|-------|-----------|----------|
| Single | success / failure | API 호출, DB 조회 |
| Completable | completed / error | 저장, 삭제, 로그 전송 |
| Maybe | success / completed / error | 캐시 조회 |

---
<br>

## Relay (RxCocoa)

Subject는 `onError`나 `onCompleted`를 받으면 **스트림이 종료**돼요

UI 바인딩에서 스트림이 끊기면 안 되기 때문에, **절대 종료되지 않는** Relay를 사용합니다

```swift
import RxCocoa

// PublishRelay = PublishSubject에서 error/completed 제거
let tapRelay = PublishRelay<Void>()
tapRelay.accept(())  // onNext 대신 accept 사용

// BehaviorRelay = BehaviorSubject에서 error/completed 제거
let nameRelay = BehaviorRelay(value: "초기값")
nameRelay.accept("새 값")
print(nameRelay.value)  // "새 값" - 현재 값 접근 가능
```

### Subject vs Relay

| | Subject | Relay |
|---|---------|-------|
| error/completed | 가능 (스트림 종료) | 불가 (종료 안 됨) |
| 값 전달 | onNext() | accept() |
| 현재 값 접근 | try? subject.value() | relay.value (BehaviorRelay) |
| 주 사용처 | 비즈니스 로직 | UI 바인딩 |

```swift
// ViewModel에서 Relay 활용
class SearchViewModel {
    let searchText = BehaviorRelay(value: "")
    let results = BehaviorRelay<[Item]>(value: [])

    // 외부에는 Observable로 노출 (읽기 전용)
    var resultObservable: Observable<[Item]> {
        return results.asObservable()
    }
}
```

---
<br>

## Error Handling

RxSwift에서 에러가 발생하면 기본적으로 **스트림이 종료**돼요

실무에서는 에러가 나도 스트림을 유지해야 하는 경우가 많습니다

### catchAndReturn - 기본값으로 대체

```swift
APIService.fetchItems()
    .catchAndReturn([])  // 에러 시 빈 배열 반환
    .subscribe(onNext: { items in
        self.updateUI(items)
    })
    .disposed(by: disposeBag)
```

### catch - 다른 Observable로 대체

```swift
APIService.fetchItems()
    .catch { error in
        print("에러 발생: \(error)")
        return CacheService.loadItems()  // 에러 시 캐시에서 로드
    }
    .subscribe(onNext: { items in
        self.updateUI(items)
    })
    .disposed(by: disposeBag)
```

### retry - 재시도

```swift
APIService.fetchItems()
    .retry(3)  // 최대 3번 재시도
    .catchAndReturn([])
    .subscribe(onNext: { items in
        self.updateUI(items)
    })
    .disposed(by: disposeBag)
```

### materialize / dematerialize

이벤트 자체를 값으로 다루고 싶을 때 사용해요

```swift
someObservable
    .materialize()  // Event<Element>로 변환 (.next, .error, .completed가 전부 onNext로 옴)
    .do(onNext: { event in
        if case .error(let error) = event {
            print("에러 로깅: \(error)")
        }
    })
    .dematerialize()  // 다시 원래 형태로
    .subscribe(onNext: { value in
        print(value)
    })
    .disposed(by: disposeBag)
```

### flatMap 내부 에러 처리 (핵심 패턴)

flatMap 안에서 에러가 나면 **외부 스트림까지 종료**돼요

이걸 방지하려면 내부에서 에러를 잡아야 합니다

```swift
// 잘못된 예 - API 에러 시 버튼 탭 스트림 자체가 죽음
button.rx.tap
    .flatMapLatest { _ in
        APIService.submit()  // 에러 → 전체 종료!
    }

// 올바른 예 - 내부에서 에러를 잡아서 외부 스트림 보호
button.rx.tap
    .flatMapLatest { _ in
        APIService.submit()
            .catch { error in
                print("실패: \(error)")
                return .empty()  // 에러를 삼키고 빈 스트림 반환
            }
    }
    .subscribe(onNext: { result in
        print("성공: \(result)")
    })
    .disposed(by: disposeBag)
```

이건 면접에서 "flatMap 안에서 에러 처리 어떻게 하세요?" 라고 물어보는 단골 질문이에요

---
<br>

## share / share(replay:) - 스트림 공유

하나의 Observable을 여러 곳에서 구독하면 **구독할 때마다 새로 실행**돼요

API 호출이라면 동일한 요청이 여러 번 나가는 셈입니다

```swift
// 문제 상황 - API 호출이 2번 발생
let request = APIService.fetchUser(id: 1)

request
    .map { $0.name }
    .bind(to: nameLabel.rx.text)
    .disposed(by: disposeBag)

request
    .map { $0.profileImageURL }
    .bind(to: profileImageView.rx.imageURL)
    .disposed(by: disposeBag)

// → fetchUser가 2번 호출됨!
```

```swift
// 해결 - share로 스트림 공유
let request = APIService.fetchUser(id: 1)
    .share(replay: 1, scope: .whileConnected)

request
    .map { $0.name }
    .bind(to: nameLabel.rx.text)
    .disposed(by: disposeBag)

request
    .map { $0.profileImageURL }
    .bind(to: profileImageView.rx.imageURL)
    .disposed(by: disposeBag)

// → fetchUser가 1번만 호출됨!
```

| 옵션 | 설명 |
|------|------|
| `share()` | 구독자 있는 동안만 공유, replay 없음 |
| `share(replay: 1)` | 최신 값 1개 버퍼링 (늦게 구독해도 받음) |
| `scope: .whileConnected` | 구독자 0이 되면 리셋 |
| `scope: .forever` | 한 번 실행되면 계속 유지 |

---
<br>

## Custom Operator / Extension 패턴

실무에서 자주 쓰는 패턴은 Extension으로 빼두면 가독성이 좋아져요

### 로딩 상태 관리

```swift
extension ObservableType {
    func trackLoading(_ loading: BehaviorRelay<Bool>) -> Observable<Element> {
        return self.do(
            onSubscribe: { loading.accept(true) },
            onDispose: { loading.accept(false) }
        )
    }
}

// 사용
let isLoading = BehaviorRelay(value: false)

APIService.fetchItems()
    .trackLoading(isLoading)
    .subscribe(onNext: { items in
        self.updateUI(items)
    })
    .disposed(by: disposeBag)

isLoading
    .bind(to: loadingIndicator.rx.isAnimating)
    .disposed(by: disposeBag)
```

### Unwrap Optional

```swift
extension ObservableType {
    func unwrap<T>() -> Observable<T> where Element == T? {
        return self.compactMap { $0 }
    }
}

// 사용
textField.rx.text  // Observable<String?>
    .unwrap()       // Observable<String>
    .subscribe(onNext: { text in
        print(text)  // 옵셔널 해제됨
    })
    .disposed(by: disposeBag)
```

---
<br>

## Input-Output 패턴 (MVVM)

RxSwift로 MVVM을 구현할 때 가장 많이 쓰이는 패턴이에요

ViewModel의 인터페이스를 Input/Output으로 명확하게 분리합니다

```swift
class LoginViewModel {

    struct Input {
        let email: Observable<String>
        let password: Observable<String>
        let loginTap: Observable<Void>
    }

    struct Output {
        let isLoginEnabled: Observable<Bool>
        let loginResult: Observable<Result<User, Error>>
    }

    func transform(input: Input) -> Output {
        let isLoginEnabled = Observable
            .combineLatest(input.email, input.password) { email, pw in
                return email.count >= 3 && pw.count >= 6
            }

        let loginResult = input.loginTap
            .withLatestFrom(Observable.combineLatest(input.email, input.password))
            .flatMapLatest { email, password in
                AuthService.login(email: email, password: password)
                    .map { Result<User, Error>.success($0) }
                    .catch { .just(.failure($0)) }
            }
            .share(replay: 1)

        return Output(
            isLoginEnabled: isLoginEnabled,
            loginResult: loginResult
        )
    }
}
```

```swift
// ViewController에서 바인딩
class LoginViewController: UIViewController {
    let viewModel = LoginViewModel()
    let disposeBag = DisposeBag()

    override func viewDidLoad() {
        super.viewDidLoad()

        let input = LoginViewModel.Input(
            email: emailField.rx.text.orEmpty.asObservable(),
            password: passwordField.rx.text.orEmpty.asObservable(),
            loginTap: loginButton.rx.tap.asObservable()
        )

        let output = viewModel.transform(input: input)

        output.isLoginEnabled
            .bind(to: loginButton.rx.isEnabled)
            .disposed(by: disposeBag)

        output.loginResult
            .observe(on: MainScheduler.instance)
            .subscribe(onNext: { [weak self] result in
                switch result {
                case .success(let user):
                    self?.navigateToHome(user: user)
                case .failure(let error):
                    self?.showError(error)
                }
            })
            .disposed(by: disposeBag)
    }
}
```

이 패턴의 장점은 ViewController가 ViewModel의 내부 로직을 전혀 몰라도 된다는 거예요

테스트할 때도 Input만 넣어주고 Output을 검증하면 끝입니다

---
<br>

## 메모리 관리 심화

### 순환 참조 방지

클로저 안에서 self를 캡처하면 순환 참조가 발생할 수 있어요

```swift
// 순환 참조 발생!
observable
    .subscribe(onNext: { value in
        self.updateUI(value)  // self → disposeBag → subscription → self
    })
    .disposed(by: disposeBag)

// 해결 - [weak self]
observable
    .subscribe(onNext: { [weak self] value in
        self?.updateUI(value)
    })
    .disposed(by: disposeBag)
```

### withUnretained - weak self 대체

매번 `[weak self]` + `guard let self` 쓰는 게 번거롭다면

```swift
observable
    .withUnretained(self)
    .subscribe(onNext: { owner, value in
        owner.updateUI(value)  // self가 해제되면 자동으로 구독 중단
    })
    .disposed(by: disposeBag)
```

### DisposeBag 교체 패턴

특정 시점에 기존 구독을 전부 해제하고 새로 구독하고 싶을 때

```swift
class PagingViewController: UIViewController {
    var disposeBag = DisposeBag()

    func refresh() {
        disposeBag = DisposeBag()  // 기존 구독 전부 해제

        // 새로운 구독 시작
        APIService.fetchItems(page: 1)
            .subscribe(onNext: { [weak self] items in
                self?.updateUI(items)
            })
            .disposed(by: disposeBag)
    }
}
```

---
<br>

## Testing

RxSwift 테스트는 **RxTest**와 **RxBlocking** 두 가지 방법이 있어요

### RxTest - TestScheduler로 시간 제어

가상 시간을 사용해서 비동기 코드를 동기적으로 테스트합니다

```swift
import RxTest

func test_검색어_디바운스() {
    let scheduler = TestScheduler(initialClock: 0)
    let disposeBag = DisposeBag()

    // 가상 시간에 이벤트 정의
    let input = scheduler.createHotObservable([
        .next(100, "S"),
        .next(200, "Sw"),
        .next(300, "Swi"),    // 300ms에 마지막 입력
        .next(700, "Swift")   // 700ms에 새 입력
    ])

    let output = scheduler.createObserver(String.self)

    input
        .debounce(.milliseconds(300), scheduler: scheduler)
        .subscribe(output)
        .disposed(by: disposeBag)

    scheduler.start()

    // debounce 300ms 적용 후 예상 결과
    XCTAssertEqual(output.events, [
        .next(600, "Swi"),     // 300 + 300ms 후 방출
        .next(1000, "Swift")   // 700 + 300ms 후 방출
    ])
}
```

### RxBlocking - 동기적으로 값 확인

간단한 스트림 테스트에 적합합니다

```swift
import RxBlocking

func test_필터링() {
    let result = try! Observable.of(1, 2, 3, 4, 5)
        .filter { $0 > 3 }
        .toBlocking()          // 블로킹 모드로 전환
        .toArray()             // 모든 값을 배열로 수집

    XCTAssertEqual(result, [4, 5])
}

func test_API_호출() {
    let result = try! viewModel.fetchUser(id: 1)
        .toBlocking(timeout: 5)  // 최대 5초 대기
        .single()                // 값 하나만 기대

    XCTAssertEqual(result.name, "욱승")
}
```

### Input-Output 패턴 테스트

```swift
func test_로그인_버튼_활성화() {
    let scheduler = TestScheduler(initialClock: 0)
    let disposeBag = DisposeBag()
    let viewModel = LoginViewModel()

    let email = scheduler.createHotObservable([
        .next(100, "ab"),       // 3자 미만
        .next(200, "abc")       // 3자 이상
    ])

    let password = scheduler.createHotObservable([
        .next(100, "12345"),    // 6자 미만
        .next(300, "123456")    // 6자 이상
    ])

    let output = viewModel.transform(input: .init(
        email: email.asObservable(),
        password: password.asObservable(),
        loginTap: .never()
    ))

    let result = scheduler.createObserver(Bool.self)

    output.isLoginEnabled
        .subscribe(result)
        .disposed(by: disposeBag)

    scheduler.start()

    // 200에서 email은 충족, password 미충족 → false
    // 300에서 둘 다 충족 → true
    XCTAssertEqual(result.events, [
        .next(100, false),
        .next(200, false),
        .next(300, true)
    ])
}
```

---
<br>

## 면접 포인트 정리

- **Traits를 왜 쓰나요?**: Observable보다 의미가 명확하고, 이벤트 타입을 컴파일 타임에 제한할 수 있음
- **Subject vs Relay**: Relay는 error/completed가 없어서 UI 바인딩에 안전. Subject는 스트림 종료가 필요한 비즈니스 로직에 사용
- **flatMap 에러 처리**: 내부에서 catch로 잡지 않으면 외부 스트림이 종료됨. 반드시 내부에서 에러를 핸들링해야 함
- **share를 언제 쓰나요?**: 하나의 Observable을 여러 곳에서 구독할 때 Side Effect 중복 실행 방지 (API 다중 호출 등)
- **메모리 누수 디버깅**: `RxSwift.Resources.total`로 현재 활성 리소스 수를 확인할 수 있음. deinit에서 확인하는 패턴 사용
- **Input-Output 패턴의 장점**: ViewModel 로직의 테스터빌리티 확보, VC와 VM의 명확한 역할 분리
- **Hot vs Cold 심화**: share()는 Cold를 Hot으로 바꾸는 역할. multicast/publish/refCount의 편의 래퍼임
