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

Languages that allow mutation make it easy to access global state and mutate it. Uncontrolled mutations of shared global state can easily cause [combinatorial explosion](https://en.wikipedia.org/wiki/Combinatorial_explosion#Computing).

But on the other hand, when used in a smart way, imperative languages can enable writing more efficient code closer to hardware.

The usual way to battle combinatorial explosion is to keep state as simple as possible, and use [unidirectional data flows](https://developer.apple.com/videos/play/wwdc2014-229) to model derived data.

This is where Rx really shines.

Rx is that sweet spot between functional and imperative worlds. It enables you to use immutable definitions and pure functions to process snapshots of mutable state in a reliable composable way.

So what are some practical examples?

### Easy integration

What if you need to create your own observable? It's pretty easy. This code is taken from RxCocoa and that's all you need to wrap HTTP requests with `URLSession`

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

### Benefits

In short, using Rx will make your code:

* Composable <- Because Rx is composition's nickname
* Reusable <- Because it's composable
* Declarative <- Because definitions are immutable and only data changes
* Understandable and concise <- Raising the level of abstraction and removing transient states
* Stable <- Because Rx code is thoroughly unit tested
* Less stateful <- Because you are modeling applications as unidirectional data flows
* Without leaks <- Because resource management is easy

### It's not all or nothing

It is usually a good idea to model as much of your application as possible using Rx.

But what if you don't know all of the operators and whether or not there even exists some operator that models your particular case?

Well, all of the Rx operators are based on math and should be intuitive.

The good news is that about 10-15 operators cover most typical use cases. And that list already includes some of the familiar ones like `map`, `filter`, `zip`, `observeOn`, ...

There is a huge list of [all Rx operators](http://reactivex.io/documentation/operators.html).

For each operator, there is a [marble diagram](http://reactivex.io/documentation/operators/retry.html) that helps to explain how it works.

But what if you need some operator that isn't on that list? Well, you can make your own operator.

What if creating that kind of operator is really hard for some reason, or you have some legacy stateful piece of code that you need to work with? Well, you've got yourself in a mess, but you can [jump out of Rx monads](GettingStarted.md#life-happens) easily, process the data, and return back into it.
