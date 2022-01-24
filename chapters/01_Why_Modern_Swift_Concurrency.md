# 1. Why Modern Swift Concurrency

Swift 5.5 에 비동기, 병렬 코드를 작성하기 위한 새로운 네이티브 모델이 추가되었습니다.

새로운 동시성 모델은 Swift 로 안전하고 성능이 뛰어난 프로그램을 작성하기 위해 필요한 것들을 제공합니다.

- 구조화된 방법으로 비동기 작업을 실행하기 위한 새로운 네이티브 문법입니다.
- 비동기, 동시성 코드를 설계하기 위한 표준 API 번들입니다.
- libdispatch 프레임워크 내의 low-level 변경으로, 모든 high-level 변경이 os 에 직접 통합됩니다.
- 안전한 동시성 코드를 만들기 위한 새로운 레벨의 컴파일러 지원입니다.

새로운 동시성 모델을 사용하려면 최신 Swift 버전을 사용하며, 특정 플랫폼 버전을 타겟으로 지정해야 합니다. 
(Xcode 13.2 이상 버전을 사용하는 경우 iOS 13 / macOS 10.15)

## 비동기, 동시성 코드 이해하기

대부분의 코드는 작성하는 방식과 동일하게 위에서 아래로 한 줄씩 실행됩니다.
동기 컨텍스트에서는 코드는 싱글 CPU 코어의 한 실행 쓰레드에서 동작합니다. 
동기 함수의 동작은 1차선 도로 위의 자동차로 상상할 수 있습니다. 
구급차 처럼 우선순위가 높은 차량이 있더라도, 다른 차량을 추월해 더 빠르게 운전할 수 없습니다.

반면 iOS 앱과 macOS 앱 본질적으로 비동기적입니다.

비동기 실행은 프로그램의 여러 요소들이 한 쓰레드에서 어떤 순서로든 실행될 수 있게 합니다. 
때로는 사용자 입력과 네트워크 연결 등의 이벤트에 따라 여러 쓰레드에서 동시에 실행될 수 있습니다.
비동기 컨텍스트에서 동일한 쓰레드를 사용해야 할 때, 함수가 실행되는 정확한 순서를 알기는 어렵습니다.

비동기 호출의 한 예로 네트워크 요청을 하고 서버가 응답할 때 컴플리션 클로저를 실행하는 것이 있습니다. 
컴플리션 콜백을 실행하기 위해 기다리는 동안, 앱은 다른 일을 처리합니다.

프로그램을 병렬로 실행하기 위해 동시성 API 를 사용합니다. 
일부 API 는 고정된 수의 작업을 동시에 실행할 수 있도록 지원하고, 다른 API 는 동시성 그룹을 시작하고 여러 동시성 작업을 허용합니다.

이는 동시성과 관련된 여러 문제들을 일으킵니다. 
예를 들어, 프로그램의 다른 부분이 서로의 실행을 블락하거나, 여러 함수가 동시에 동일한 변수에 엑세스하여 문제를 일으키는 데이터 레이스가 발생합니다.

하지만 동시성을 조심해서 사용하면 멀티 CPU 코어에서 동시에 여러 함수를 실행해 프로그램이 더 빠르게 동작하도록 만들 수 있습니다. 
이는 n차선 도로에서 운전자들이 더 빠르게 이동할 수 있는 것과 유사합니다. 
차선이 여러개이기 때문에 더 빠른 차량이 느린 차들을 앞서갈 수 있고, 구급차 같은 우선순위가 높은 차량은 비상 차선을 사용할 수 있습니다.

구급차 처럼 우선 순위가 높은 작업은 낮은 작업의 대기열을 점프할 수 있어서, UI 업데이트를 위한 메인 쓰레드를 블락하지 않습니다. 
예를 들면 서버에서 여러 이미지를 동시에 다운로드하고, 썸네일 크기로 축소해서, 캐시에 저장해야 하는 사진 검색 사례가 있습니다.

## 기존의 동시성 프로그래밍 알아보기

Swift 5.5 이전 버전에서는 아래의 도구들을 이용해 비동기 코드를 실행했습니다.

1. GCD 의 DispatchQueue
2. Operation, Thread 같은 metal 에 가까운 오래된 API
3. C 기반의 pthread 라이브러리를 직접 사용

이 API 들은 모두 프로그래밍 언어에 의존하지 않는 POSIX thread 를 사용합니다. 각 실행 흐름은 쓰레드이고, 여러 쓰레드가 동시에 겹쳐서 실행될 수도 있습니다.

Operation, Thread 같은 Thread Wrapper 의 경우 쓰레드를 수동으로 관리해야 해서 오류가 발생하기 쉬웠습니다.

GCD 의 큐 기반 모델은 잘 작동했지만, 간혹 이런 문제를 일으키기도 했습니다.

- Thread explosion : 여러 병렬 쓰레드를 생성하면 쓰레드 사이를 계속 전환해야 합니다. 결과적으로 앱 속도가 느려집니다.
- Priority inversion : 같은 큐의 우선순위가 낮은 작업이 높은 우선순위의 작업의 실행을 블락합니다.
- Lack of execution hierarchy : 실행중인 작업에 접근하거나 취소하는 것이 어려웠습니다. 이는 결과 반환하는 것도 복잡하게 만들었습니다.

이런 단점을 해결하기 위해 Swift 는 새로운 동시성 모델을 도입하였습니다.

## 새로운 Swift 동시성 모델

새로운 동시성 모델은 언어 문법, Swift 런타임, Xcode 와 강하게 통합되며, 이는 개발자를 위해 쓰레드 개념을 추상화합니다.

새로운 주요 기능은 아래와 같습니다.

1. Cooperative thread pool
2. async / await
3. Structured concurrency
4. Context-aware code compilation

### 1. Cooperative thread pool

새로운 모델은 쓰레드 풀을 투명하게 관리해 사용가능한 CPU 코어 수를 초과하지 않도록 만듭니다.

이런 방식으로 런타임에서 쓰레드를 생성/제거하거나 비용이 비싼 쓰레드 전환을 계속 수행하지 않아도 됩니다.

대신 코드가 중단되었다가 풀의 사용 가능한 쓰레드에서 빠르개 재개될 수 있습니다.

### 2. async / await

async / await 문법을 통해 컴파일러와 런타임은 일부 코드가 중단되었다가 다시 재개될 수 있다는 것을 알 수 있습니다.

런타임에서 이를 원활하게 처리하기 때문에, 쓰레드와 코어에 대한 걱정을 하지 않아도 됩니다.

추가로 탈출 클로저를 사용하지 않기 때문에 self 나 다른 변수를 약하게 캡처할 필요가 없어집니다.

### 3. Structured concurrency

각 비동기 작업은 부모 Task와 주어진 실행 우선순위를 가지고 계층에 속하게 됩니다.

이 계층을 통해 런타임은 부모 작업이 취소될 때 모든 하위작업을 취소할 수 있습니다.

또한 런타임이 부모 작업이 완료되기 전에 모든 자식 작업을 완료하도록 기다리게 합니다.

이는 계층 내에서 높은 우선순위의 작업이 낮은 우선순위 작업보다 먼저 실행될 수 있다는 장점과 확실한 결과를 제공합니다.

### 4. Context-aware code compilation

컴파일러는 주어진 코드가 비동기로 실행되는지 트래킹합니다.

비동기로 실행된다면 공유 상태(shared state)를 변경하는 것처럼 안전하지 않은 코드를 작성할 수 없습니다.

이런 높은 수준의 컴파일러 인식은 Actor 같은 새로운 기능을 가능하게 합니다.

> Actor 는 비동기/동기 접근에서 안전하지 않은 코드를 작성하기 어렵게 만들어 데이터가 손상되는 것을 방지합니다.

## async / await 사용하기

- [예제 코드](https://github.com/raywenderlich/mcon-materials/tree/editions/1.0/00-book-server)

```swift
func availableSymbols() async throws -> [String] {
  guard let url = URL(string: "http://localhost:8080/littlejohn/symbols") else {
    throw "url error"
  }
}
```

함수 정의에 쓰이는 async 키워드는 컴파일러가 코드가 비동기 컨텍스트에서 실행된다는 것을 알 수 있게 합니다.

이것은 코드가 중단됐다가 재개될 수 있다는 것을 의미합니다. 

또한 함수가 완료되는 시간과 무관하게 동기 메서드와 유사한 값을 반환합니다. (예제에서는 [String])

```swift
let (data, response) = try await URLSession.shared.data(from: url)
```

foo 함수 하단에 URLSession 을 호출해 데이터를 가져오는 코드를 추가합니다.

비동기 메서드 `URLSession.data(from:delegate:)` 를 호출하면 foo 함수가 중단되었다가 서버에서 데이터를 가져올 때 재개됩니다.

await 을 사용하여 런타임에 중단 포인트를 줄 수 있습니다. 함수를 중지하고, 먼저 실행할 다른 Task 를 실행한 뒤 코드를 계속 진행합니다.

```swift
guard (response as? HTTPURLResponse)?.statusCode == 200 else {
  throw "response error"
}

return try JSONDecoder().decode([String].self, from: data)
```

다음으로 위 코드를 실행해 서버 응답 코드가 200인지 확인한 뒤 JSON 을 디코딩하여 반환합니다.

## SwiftUI 에서 async / await 사용하기

- [예제 코드](https://github.com/raywenderlich/mcon-materials/blob/editions/1.0/01-hello-modern-concurrency/projects/final/LittleJohn/SymbolListView.swift)

```swift
.task {
  do {
    symbols = try await model.availableSymbols()
  } catch {
    errorMessage = error.localizedDescription
  }
}
```

SwiftUI 에서는 `task(priority:_:)` 를 사용해서 비동기 작업을 실행할 수 있습니다. 
task 는 onAppear 와 마찬가지로 뷰가 화면이 표시될 때 호출됩니다.

위 예제 처럼 try 와 await 을 활용해 비동기적으로 값을 반환받고 실패할 경우 에러를 핸들링할 수 있습니다.

## Asynchronous sequence 사용하기

- [예제 코드](https://github.com/raywenderlich/mcon-materials/blob/editions/1.0/01-hello-modern-concurrency/projects/final/LittleJohn/TickerView.swift)

```swift
let (stream, response) = try await liveURLSession.bytes(from: url)

for try await line in stream.lines {
  let sortedSymbols = try JSONDecoder()
    .decode([Stock].self, from: Data(line.utf8))
    .sorted(by: { $0.name < $1.name })
  
  tickerSymbols = sortedSymbols
}
```

stream 은 서버가 응답으로 보내는 바이트 시퀀스입니다. 

lines 는 응답의 text line 을 하나씩 제공하는 시퀀스를 추상화한 것 입니다. lines 는 반복문을 거쳐 JSON 으로 디코딩합니다. (for 문 내에 작성된 코드)

tickerSymbols 가 백그라운드 쓰레드에서 변경되면서 보라색 경고를 볼 수 있습니다. (UI 는 메인 쓰레드에서 동작해야 함)

## 메인 쓰레드에서 UI 변경하기

비동기 작업을 실행하는 컨텍스트에서 tickerSymbols 를 변경하면, 코드는 풀에 있는 임의 쓰레드에서 실행됩니다.

UI 업데이트를 담당하는 상태를 변경하는 경우, 아래처럼 동작시킬 수 있습니다.

```swift
// AS-IS
tickerSymbols = sortedSymbols

// TO-BE
await MainActor.run {
  tickerSymbols = sortedSymbols
}
```

MainActor 는 코드를 메인 쓰레드에서 실행시킵니다. (`MainActor.run(_:)`을 사용)

## 동시성 작업 취소하기

- [예제 코드](https://github.com/raywenderlich/mcon-materials/blob/editions/1.0/01-hello-modern-concurrency/projects/final/LittleJohn/TickerView.swift)

Modern Swift Concurrency 의 장점 중 하나는 구조화된 방법으로 동시성 코드가 실행되는 것입니다.

Task 들은 스트릭트한 계층에서 실행되어 런타임은 누가 Task 의 부모 Task 인지, 어떤 새로운 Task 들이 상속되어야 하는지 알 수 있습니다.

```swift
.task {
  do {
    try await model.startTicker(selectedSymbols)
  } catch {
    if let error = error as? URLError,
           error.code == .cancelled {
      return
    }
    
    lastErrorMessage = error.localizedDescription
  }
}
```

위 코드의 `task(_:)` 를 살펴보면 `startTicker(_:)` 를 비동기로 호출하고 이는 비동기 시퀀스를 반환합니다.

await 키워드가 있는 각 지점에서 매번 쓰레드가 변경될 수 있습니다. 전체 프로세스는 `task(_:)` 내부에서 시작하기 때문에 비동기 Task 는 실행 쓰레드나 중단 상태와 무관하게 다른 Task 들의 부모 Task 입니다.

SwiftUI 의 `task(_:)` 는 뷰가 사라질 때 비동기 코드를 취소합니다. 동작을 실행한 뒤 백버튼을 눌러 동작이 취소되는 것을 확인할 수 있습니다.

취소에 대한 에러를 처리해야 하는 경우 catch 에서 처리를 해주면 됩니다. 예제의 경우 URLError 의 code 가 cancelled 인 것을 확인하였습니다. 

Task.sleep 과 같은 최신 비동기 API 에는 CancellationError 에러를 발생시킵니다. 커스텀 에러를 발생시키는 경우 URLSession 처럼 취소와 관련된 에러코드가 존재합니다.

## 스터디에 사용된 추가 자료

- [Swift Concurrency 최소 버전 이야기](https://forums.swift.org/t/will-swift-concurrency-deploy-back-to-older-oss/49370/77)
- [Concurrency Asynchronous Functions](https://forums.swift.org/t/concurrency-asynchronous-functions/41619/43)
- [Async Sequence Proposal](https://github.com/apple/swift-evolution/blob/main/proposals/0298-asyncsequence.md)
- [WWDC-Swift concurrency: Behind the scenes](https://developer.apple.com/videos/play/wwdc2021/10254/)
- [Apple Doc - AsyncStream](https://developer.apple.com/documentation/swift/asyncstream)
