# Getting Started With async/await

이번 챕터에서는 실제 async/await 구문과 이것이 어떻게 비동기 실행을 조정하는지 알아보겠습니다.

추가로 Task type 과 비동기 실행 컨텍스트를 어떻게 생성하는지 알 수 있습니다.

먼저 Swift 5.5 (async/await) 이전의 동시성 프로그래밍에 대해 알아봅시다.

## async/await 이전의 비동기 프로그래밍

```swift
final class API {
  func fetch(completion: @escaping (Data) -> Void) {
    URLSession.shared
      .dataTask(
        with: URL(string: "https://test.com/test")!
      ) { data, response, error in
        completion(data)
      }
      .resume()
  }
}

final class ViewModel {
  let api = API()
  
  var data: Data?
  
  func fetch() {
    api.fetch { [weak self] data in
      self?.data = data
    }
  }
}
```

위 예제는 API 를 호출하는 코드입니다.

이는 간단하지만, 의도를 모호하게 만들며, 여러 에러를 만들어 낼 수 있습니다. 
ex) completion 호출에 대해 검증할 수 없습니다.

Swift 는 Objective-C 를 위해 디자인된 GCD 에 의존했기 때문에, 처음부터 언어 디자인에 비동기성을 통합하기 어려웠습니다. Objective-C 의 경우 언어가 시작된 수년 후인 iOS 4 에 블록(Swift 의 클로저와 유사함)이 도입되었습니다.

다시 위의 예제를 살펴보겠습니다.

1. 먼저 컴파일러는 `fetch()` 내에서 completion 의 호출횟수를 알 수 있는 방법이 없습니다. 따라서 메모리 사용과 수명을 최적화할 수 없습니다.
2. 해당 코드를 사용할 때 약한참조(weak)를 이용해 메모리를 직접 관리해야 합니다.
3. 컴파일러는 에러를 핸들링했는지 알 수 없습니다. completion 핸들러를 호출하지 않거나 에러를 핸들링하지 않으면 문제가 발생할 수 있습니다.

Swift 의 Modern Concurrency Model 은 컴파일러와 런타임 모두와 긴밀하게 동작해 위의 문제를 포함한 많은 문제들을 해결합니다.

Modern Concurrency Model 은 아래의 세가지 도구를 제공합니다.

- async : 메서드 혹은 함수가 비동기임을 나타냅니다. 이를 이용해 비동기 메서드가 결과를 반환할 때까지 실행을 중단할 수 있습니다.
- await : 코드가 async 메서드 혹은 함수가 반환되기 전까지 실행을 중지할 수 있음을 나타냅니다.
- Task : 비동기 작업의 단위입니다. Task 가 완료되기를 기다리거나, 완료되기 전에 취소할 수 있습니다.

위 예제를 Modern Concurrency 를 이용해 다시 작성해보겠습니다.

```swift
final class API {
  func fetch() async throws -> Data {
    let (data, _) = try await URLSession.shared.data(
      from: URL(string: "https://test.com/test")!
    )
    
    return data
  }
}

final class ViewModel {
  let api = API()
  
  var data: Data?
  
  func fetch() {
    Task {
      data = try await api.fetch()
    }
  }
}
```

위 코드는 컴파일러와 런타임에게 더 명확합니다.

- `fetch()` 는 실행을 중단했다 재개할 수 있는 비동기 함수입니다. async 를 사용하여 표시합니다.
- `fetch()` 는 데이터를 반환하거나 에러를 throw 합니다. 컴파일 타임에 확인이 가능해서 오동작을 방지할 수 있습니다.
- Task 는 주어진 클로저를 비동기 컨텍스트에서 실행해서, 컴파일러는 클로저 내에서 쓰기(변경)에 안전한지 알 수 있습니다.
- await 을 사용해 런타임에게 비동기 함수를 호출할 때마다 코드를 중단하거나 취소할 수 있는 기회를 제공합니다. 이는 시스템이 현재 Task queue 의 우선순위를 지속적으로 변경할 수 있게 합니다.

## 코드를 partial tasks 로 분리하기

CPU 코어와 메모리 처럼 공유 자원을 최적화하기 위해, Swift 는 코드를 partial task 혹은 partials 로 불리는 논리 단위로 분리합니다.

<img src="./images/02-partial-task-1.png">

Swift 런타임은 비동기 실행을 위해 이 조각들을 각각 스케줄링합니다. 각 partial task 가 완료되면 시스템은 보류된 task의 우선순위와 시스템 부하에 따라 코드를 계속할지 다른 task 를 실행할지 결정합니다.

따라서 await 어노테이션이 붙은 partial task 들은 시스템 재량에 따라 다른 쓰레드에서 실행될 수 있습니다. 또한 await 후에 앱의 상태를 가정해서는 안됩니다. 작성된 코드는 차례대로 나타나지만 실행시간이 많이 차이날 수도 있습니다. task 를 기다리는건 임의의 시간이 걸리며 그 사이에 앱의 상태가 크게 변경될 수 있습니다.

요약하자면 async/await 은 간단하지만 강력한 구문입니다. 이는 컴파일러가 안전하고 견고한 코드를 작성하도록 가이드하고, 런타임이 공유 시스템 자원의 사용을 최적화하도록 합니다.

### partial tasks 실행하기

async, await, let 와 같은 키워드를 사용하는 것은 의도를 명확하게 표현합니다. 동시성 모델의 기반은 비동기 코드를 Executor 에서 실행하는 partial tasks 로 나누는 것을 중심으로 다룹니다. 

<img src="./images/02-partial-task-2.png">

Executor 는 GCD queue 와 비슷하지만, 더 강력하고 low-level 입니다. 그리고 Executor 는 작업을 빠르게 실행하고, 실행 순서와 쓰레드 관리 같은 복잡함을 완전히 숨길 수 있습니다.

## Task 의 수명 관리하기

Modern Concurrency 의 중요한 신규 기능 중 하나는 비동기 코드의 수명을 관리하는 시스템의 능력입니다.

기존 멀티 쓰레드 API 의 가장 큰 단점은 비동기 코드가 시작되면 그 코드가 제어를 포기하기 전까지, 시스템이 CPU 코어를 회수할 수 없었다는 점입니다. 이로 인해 특정 작업이 더이상 필요하지 않아도 리소스를 소비하고, 필요하지 않은 작업을 수행합니다.

서버에서 컨텐츠를 가져오는 서비스가 좋은 예제입니다. 서비스를 두 번 호출하면 시스템은 첫번째 호출이 사용한 리소스를 회수할하는 자동 메커니즘이 없어, 불필요한 리소스를 낭비하게 됩니다.

새로운 비동기 모델은 코드를 부분으로 나누어 런타임에서 체크하는 중단 지점을 제공합니다. 
이는 시스템이 코드를 정지시키거나 취소할 수 있는 기회를 줍니다.

덕분에 주어진 작업을 취소할 때 런타임은 비동기 계층으로 이동할 수 있고, 하위 작업을 취소할 수 있습니다.

<img src="./images/02-cancellation.png">

하지만 중단 지점 없이 긴 계산을 수행하는 작업이 있으면 어떨까요? 이런 경우 Swift 는 현재 작업이 취소되었는지 알 수 있는 API 를 제공합니다. 이 경우 수동으로 실행을 포기할 수 있습니다.

마지막으로 중단 지점은 에러를 위한 탈출 경로를 제공해 코드에서 에러를 캐치하고 핸들링하는 코드로 계층을 끌어올립니다.

<img src="./images/02-async-errors.png">

새로운 비동기 모델은 잘 알려진 throw 함수 등을 이용해 동기 함수가 가진 구조와 유사하게 에러를 핸들링 할 수 있도록 제공합니다. 또한 task 가 에러를 throw 하는 즉시 메모리를 해제하도록 최적화 되었습니다.

Modern Swift Concurrency Model 에서 반복되는 토픽은 안전함(safety), 최적화된 리소스 사용(optimized resource usage), 최소 구문(minimal syntax) 입니다. 뒤에서는 새로운 API 에 대해 자세히 알아보고 사용해보겠습니다.



---

이미지 리소스 출처 및 원문

- https://www.raywenderlich.com/books/modern-concurrency-in-swift/v1.0/chapters/2-getting-started-with-async-await