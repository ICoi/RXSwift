## 왜 RxSwift를 사용하는 지(Why)

**Rx는 선언형 방식으로 앱을 만들 수 있게합니다.**

### 바인딩(Bindings)

```swift
Observable.combineLatest(firstName.rx.text, lastName.rx.text) { $0 + " " + $1 }
    .map { "Greetings, \($0)" }
    .bind(to: greetingLabel.rx.text)
```

이 방식은 'UITableView'나 'UICollectionView'에서도 활용 할 수 있습니다.

```swift
viewModel
    .rows
    .bind(to: resultsTableView.rx.items(cellIdentifier: "WikipediaSearchCell", cellType: WikipediaSearchCell.self)) { (_, viewModel, cell) in
        cell.title = viewModel.title
        cell.url = viewModel.url
    }
    .disposed(by: disposeBag)
```
**간단한 바인딩 로직을 수행할 때에도 항상 `disposed(by:disposeBag)`을 이용하여 메모리 해제시킬 것을 권장합니다.**

### 재시도(Retries)

모든 API 호출은 항상 성공으로 응답하기 때문에 실패 상황에 대한 대응이 필요합니다. 아래의 API 함수로 예를 들어 봅시다.

```swift
func doSomethingIncredible(forWho: String) throws -> IncredibleThing
```

이러한 함수에서는 실패시 재시도 로직을 구현하기 위해선 [exponential backoffs](https://en.wikipedia.org/wiki/Exponential_backoff)와 같은 복잡한 모델을 도입하는 코드 작업이 필요합니다. 재사용 될 지 알 수 없고, 신경쓰고 싶지도 않을 복잡한 로직을 이해하는 번거로운 작업이 동반되는 것입니다.

Rx는 아래와 같이 모든 연산자(operation)에 적용할 수 있는 재시도 모델을 제공합니다.

```swift
doSomethingIncredible("me")
    .retry(3)
```

쉽게 재시도 연산자(retry operation)를 추가 할 수 있습니다.

### 델리게이트(Delegates)

아래와 같이 진부하고 가독성 낮은 방식 대신

```swift
public func scrollViewDidScroll(scrollView: UIScrollView) { [weak self] // what scroll view is this bound to?
    self?.leftPositionConstraint.constant = scrollView.contentOffset.x
}
```

아래의 방식으로 작성합니다

```swift
self.resultsTableView
    .rx.contentOffset
    .map { $0.x }
    .bind(to: self.leftPositionConstraint.rx.constant)
```

### KVO (Key-Value Observing)


```
`TickTock` 은 key value observer에 등록되어있으면 메모리 해제 되지 않습니다. Observation 정보에 누수가 발생하여 다른 object에 결합되는 실수가 발생 할 수 있습니다.
```

혹은

```objc
-(void)observeValueForKeyPath:(NSString *)keyPath
                     ofObject:(id)object
                       change:(NSDictionary *)change
                      context:(void *)context
```

위의 두가지 방식 대신에 [`rx.observe` 와 `rx.observeWeakly`](GettingStarted.md#kvo) 를 사용하세요.

아래에 사용 방법에 대한 예시가 작성되어 있습니다.

```swift
view.rx.observe(CGRect.self, "frame")
    .subscribe(onNext: { frame in
        print("Got new frame \(frame)")
    })
    .disposed(by: disposeBag)
```

or

```swift
someSuspiciousViewController
    .rx.observeWeakly(Bool.self, "behavingOk")
    .subscribe(onNext: { behavingOk in
        print("Cats can purr? \(behavingOk)")
    })
    .disposed(by: disposeBag)
```

### 노티피케이션(Notifications)

기존의 사용방식을 대신하여

```swift
@available(iOS 4.0, *)
public func addObserverForName(name: String?, object obj: AnyObject?, queue: NSOperationQueue?, usingBlock block: (NSNotification) -> Void) -> NSObjectProtocol
```

아래와 같은 방식으로 작성합니다.

```swift
NotificationCenter.default
    .rx.notification(NSNotification.Name.UITextViewTextDidBeginEditing, object: myTextView)
    .map {  /*do something with data*/ }
    ....
```

### 임시 상태(Transient state. 이하, Transient state)

비동기 프로그래밍 환경에서 transient state를 꾸현하는 것은 많은 문제를 동반합니다. Rx 없이 자동 완성 검색 박스(autocomplete search box) 구현한다고 생각해봅시다.

가장 먼저 해결해야할 문제는 `abc`를 입력하기 위해 `c` 를 입력하고 `ab`입력(pending request)을 기다리고 있던 중 기다리고 있던 요청(pending request)을 취소하는 경우를 어떻게 구현 할 지입니다. 쉽게 해결 하기 위해서 별도의 변수를 생성하여 pending request 정보를 가지고 있도록 구현 할 수 있습니다.

다른 문제는 요청이 실패하는 경우 복잡한 재시도 로직을 구현해야 한다는 것 입니다. 재시도 횟수를 저장하는 별도의 변수를 할당하고, 값을 제어하도록 구현 할 수 있습니다.

유저가 긴 문장을 입력하는 경우 매번 요청을 보내는 것은 서버에 부담을 줄 수 있기 때문에, 요청을 전달 하기 전 약간의 딜레이가 필요한 경우도 있습니다. 별도의 타이머(timer) 변수를 생성해야 할 것 같습니다.

검색 수행 중, 혹은 검색 실패 (수차례 재시도 하였으나 결국 실패)와 같은 상황에서 UI 노출하는 방법에 대한 고민도 필요합니다.

위와 같은 길고 복잡한 과정을 Rx에서는 아래와 같이 간결하게 구현 가능합니다.

```swift
searchTextField.rx.text
    .throttle(.milliseconds(300), scheduler: MainScheduler.instance)
    .distinctUntilChanged()
    .flatMapLatest { query in
        API.getSearchResults(query)
            .retry(3)
            .startWith([]) // clears results on new search term
            .catchErrorJustReturn([])
    }
    .subscribe(onNext: { results in
      // bind to ui
    })
    .disposed(by: disposeBag)
```

별도의 flag나 fields 추가 필요 없이 모든 복잡한 로직을 구현 할 수 있습니다.

### Compositional disposal(관련 로직 처리)

Table view에 blur 처리 된 이미지를 노출하려고 한다고 가정해봅시다. 이미지를 URL로 부터 다운로드 받고, 복호화 한 다음 blur 처리를 진행해야 합니다.

네트워크 통신이나 blur 처리를 위한 processor 자원은 비싸므로, cell이 화면 밖으로 벗어나는 경우 모든 프로세를 중단 시키는 것이 좋습니다.

또한, 유저가 매우 빠르게 swipe 하는 경우 요청과 취소가 빈번하게 일어나므로 cell이 화면 영역에 보여지는 즉시 프로세스를 수행 할 필요는 없을 것 같습니다.

또한, blur 처리에 소모되는 자원은 비싸므로 동시에 처리되는 image 갯수를 제한하는 것도 좋을 것 같습니다.

이 모든 것을 Rx를 활용하면 아래와 같이 구현 가능합니다.

```swift
// this is a conceptual solution
let imageSubscription = imageURLs
    .throttle(.milliseconds(200), scheduler: MainScheduler.instance)
    .flatMapLatest { imageURL in
        API.fetchImage(imageURL)
    }
    .observeOn(operationScheduler)
    .map { imageData in
        return decodeAndBlurImage(imageData)
    }
    .observeOn(MainScheduler.instance)
    .subscribe(onNext: { blurredImage in
        imageView.image = blurredImage
    })
    .disposed(by: reuseDisposeBag)
```

이 코드는 모든 로직을 처리하고, `imageSubscription`이 dispose 될 때, 모든 관련된 비동기 로직을 취소하고 UI에 관련 이미지를 노출하도록 처리됩니다.

### 네트워크 요청 합치기(Aggregating network requests)

2개의 별도의 요청을 전송하고 모든 요청이 끝났을때 결과를 합치려면 어떻게 처리해야할까?

답은 `zip` 연산자(operator)에 있습니다.

```swift
let userRequest: Observable<User> = API.getUser("me")
let friendsRequest: Observable<[Friend]> = API.getFriends("me")

Observable.zip(userRequest, friendsRequest) { user, friends in
    return (user, friends)
}
.subscribe(onNext: { user, friends in
    // bind them to the user interface
})
.disposed(by: disposeBag)
```
만약 API의 결과는 background thread에서 전달되고 결과 바인딩는 UI Main 스레드에서 수행되야한다면 어떻게 처리해야할까요? `observeOn`을 사용하면 됩니다.

```swift
let userRequest: Observable<User> = API.getUser("me")
let friendsRequest: Observable<[Friend]> = API.getFriends("me")

Observable.zip(userRequest, friendsRequest) { user, friends in
    return (user, friends)
}
.observeOn(MainScheduler.instance)
.subscribe(onNext: { user, friends in
    // bind them to the user interface
})
.disposed(by: disposeBag)
```

이외에도 많은 상황에서 Rx는 뛰어난 방법을 제공합니다.


### 상태(State)


변화를 허용하는 언어는 글로벌 상태에 쉽게 접근하고 형태를 변화할 수 있게 합니다. 제어되지 않는 글로벌 상태의 변화는 combinatorial explosion을 발생 시킬 수 있습니다.
> combinatorial explosion 이란?
> Z개의 상태를 나타낼 수 있는 타입을 활용하여 n개의 변수를 만들어야 Z^n개의 상태를 표현할 수 있는 것

반면에, 명령형 언어 (imperative languages)는 코드를 좀 더 효율적으로 하드웨어에 친숙하게 작성할 수 있게 합니다.

combinatorial explosion을 대응하기 위해 가장 쉽고 가능한 방법은 단방향 데이터가 진행되도록 (unidirectional data flows) 데이터 모델을 설계하는 것 입니다.

여기서 Rx의 장점이 발휘됩니다.

Rx는 함수형과 반응형의 중간쯤에 있습니다. Rx는 변화하는 상태를 신뢰할 수 있고 조합 가능한 방법으로 처리하기 위해 반응형 정의(imuutable definitions)와 순수 함수(pure functions)을 사용합니다.

그러면 실제로는 어떻게 사용될까요?

### Easy integration 이식이 쉽다.

Observable을 사용하기 위해선 어덯게 해야할까요? 방법은 매우 쉽습니다. 아래는 URLSession을 사용하여 HTTP 요청을 진행하는 RxCocoa의 코드 일부입니다.

```swift
extension Reactive where Base: URLSession {
    public func response(request: URLRequest) -> Observable<(Data, HTTPURLResponse)> {
        return Observable.create { observer in
            let task = self.base.dataTask(with: request) { (data, response, error) in
            
                guard let response = response, let data = data else {
                    observer.on(.error(error ?? RxCocoaURLError.unknown))
                    return
                }

                guard let httpResponse = response as? HTTPURLResponse else {
                    observer.on(.error(RxCocoaURLError.nonHTTPResponse(response: response)))
                    return
                }

                observer.on(.next(data, httpResponse))
                observer.on(.completed)
            }

            task.resume()

            return Disposables.create(with: task.cancel)
        }
    }
}
```

### Benefits 장점

요악하자면, Rx는 당신의 코드를\
* 조합할 수 있음(Composable) <- Rx는 구성품들의 별칭 이므로
* 재사용 할 수 있음(Resuable) <- 조합 할 수 있으므로
* 선언적임(Declarative) <- 정의를 변경하지 않고 데이터 만 변경되므로 
* 이해하기 쉽고 간결한(Understandable and concise) <- 추상화 레벨이 증가하고 임시 상태에 대한 값들이 필요 없으므로
* 안정적인(Stable) <- Rx 코드는 전부 unit 테스트가 가능하므로
* 상태값에 대한 내용이 줄어듬(Less stateful) <- 단방향 데이터 흐름의 방식으로 앱을 모델링하므로
* 결함이 없는(Without leask) <- 리소스 관리가 쉬우므로

하게 만들어 줍니다.

### It's not all or nothing

Application을 Rx를 최대한 활용하는 것은 좋은 생각이다.

그러나, 모든 함수를 알수는 없으며, 심지어 특정 케이스에 적합한 모델을 구현 할 수 있는 함수가 존재하는지 어떻게 알아야할까?

Rx의 모든 함수들은 수학에 기반을 두고 있으며, 이해하기 쉽게 되어있다.

당신에게 희소식은, 10~15개의 함수로 대부분의 상황에 대해서는 대응 할 수 있다는 점이다. `map`, `filter`, `zip`, `observeOn`과 같이 위에 언급된 것들이 자주 사용된다.

[여기](http://reactivex.io/documentation/operators.html)에 Rx의 모든 함수에 대한 내용이 정리되어있다. 

각각의 함수에 대해서는 [marble diagram](http://reactivex.io/documentation/operators/retry.html)을 통해 어떻게 동작하는지 확인 할 수 있다.

만약 리스트에 당신이 원하는 함수가 없다면? 직접 만들어서 사용하여도 됩니다.

함수를 만드는 것이 어렵건나, 이미 상태값들로 가득찬 레거시 코드를 작업해야한다면 어떻게 할까요? 복잡하겠지만 Rx의 철학을 벗어나 쉽고, 데이터를 처리하여 값을 리턴해주도록 구현해도 됩니다. [관련자료](GettingStarted.md#lie-happens)
