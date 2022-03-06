# AsyncSequence & Intermediate Task

## AsyncSequence 알아보기

AsyncSequence 는 Sequence 와 유사하게 비동기 요소를 생성할 수 있는 프로토콜 입니다.

Sequence 와 동일하지만, 다음 요소를 즉시 사용할 수 없기 때문에 기다려야 합니다.

```swift
for try await item in asyncSequence {
  ...
}
```

AsyncSequence 를 사용한 일반적인 코드입니다. await 을 사용해 for 문안의 Sequence 를 반복합니다. (throw 처리가 있는 경우 try 를 사용합니다) 각 루프를 반복할 때 다음 값을 얻기 위해 중단됩니다.

```swift
var iterator = asyncSequence.makeAsyncIterator()

while let item = try await iterator.next() {
  ...
}
```

while 루프 문과도 사용할 수 있습니다. interactor 를 만들고 Sequence 가 끝날 때까지 await 을 사용해 반복적으로 next() 를 호출합니다.

```swift
for await item in asyncSequence
  .dropFirst(5)
  .prefix(10)
  .filter { $0 > 10 }
  .map { "Item: \($0)" } {
    ...
  }
```

표준 Sequence 메서드인 `dropFirst(_:)`, `prefix(_:)`, `filter(_:)` 등을 사용할 수 있습니다.

AsyncSequence 를 따르는 커스텀 Sequence 타입을 만들거나, AsyncStream 을 활용해 기존 코드를 AsyncSequence 로 변경할 수 있습니다.

## AsyncSequence 시작하기

이전 챕터에서는 파일을 한 번에 가져온 뒤 preview 를 보여주는 동작을 구현했습니다.

이번 챕터에서는 파일이 다운로드될 때, 점진적으로 UI 를 업데이트하도록 구현합니다.

파일을 서버에서 byte AsyncSequence 로 읽어 프로그레스바를 업데이트할 수 있습니다.

### Asynchronous sequence 추가하기

[SuperStorageModel.swift](https://github.com/raywenderlich/mcon-materials/blob/editions/1.0/03-async-sequences/projects/starter/SuperStorage/Model/SuperStorageModel.swift) 의 `downloadWithProgress(fileName:name:size:offset:)` 를 확인합니다.

위 코드에는 URL 을 만드는 코드와 `addDownload(name:)` 를 호출해 화면에 다운로드를 추가하는 코드가 작성되어 있습니다.

이 곳에 asynchronous sequence 를 추가하기 앞서 먼저 URLSession 의 API 에 대해 알아봅시다.

```swift
guard let url = URL(string: "test.com") else { return }

let result = try await URLSession.shared.data(from: url)
```

`data(for:delegate:)` 메서드를 사용하면 Data 전체를 불러와 반환합니다.

```swift
guard let url = URL(string: "test.com") else { return }

let result = try await URLSession.shared.bytes(for: urlRequest)
```

반면 `bytes(for:delegate:)` 메서드를 사용하면 `URLSession.AsyncBytes` 를 반환합니다.

`URLSession.AsyncBytes` 는 sequence 로, URL 요청에서 받은 byte 를 비동기적으로 제공합니다.

서버는 HTTP 프로토콜을 통해 부분 요청을 지원할 수 있습니다. 서버가 이를 지원하는 경우, 전체 응답을 한 번에 받지 않고 응답의 바이트 범위를 반환하도록 요청할 수 있습니다. 

앱에서 부분 요청과 일반 요청을 모두 지원하는 예제를 작성해봅시다.

```swift
guard let url = URL(string: "http://localhost:8080/files/download?\(fileName)") else {
  throw "Could not create the URL."
}

let result: (downloadStream: URLSession.AsyncBytes, response: URLResponse)

if let offset = offset {
  // 부분 요청
  let urlRequest = URLRequest(url: url, offset: offset, length: size)
  
  result = try await URLSession.shared.bytes(for: urlRequest)
  
  guard (result.response as? HTTPURLResponse)?.statusCode == 206 else {
    throw "The server responded with an error."
  }
} else {
  // 전체 요청
  result = try await URLSession.shared.bytes(from: url)
}
```

offset 이 있는 경우 그를 활용해 URLRequest 를 만들어 `URLSession.bytes(for:delegate:)` 에 사용합니다.

그 다음 status code 가 성공적인 부분 응답을 나타내는 206 인지 확인합니다.

부분 요청, 일반 요청 모두 `URLSession.AsyncBytes` 를 사용할 수 있습니다. offset 이 없는 경우 `URLSession.bytes(from:delegate:)` 를 사용합니다.

### ByteAccumulator 사용하기

이제 [ByteAccumulator](https://github.com/raywenderlich/mcon-materials/blob/editions/1.0/03-async-sequences/projects/starter/SuperStorage/Model/ByteAccumulator.swift) 라는 타입을 사용해 sequence 로 부터 batches of byte 를 불러올 수 있습니다.

ByteAccumulator 를 사용하기 위해 `downloadWithProgress(fileName:name:size:offset:) ` 의 반환 코드 이전에 아래의 코드를 추가합니다.

```swift
var asyncDownloadIterator = result.downloadStream.makeAsyncIterator()
```

AsyncSequence 는 sequence 에 대해 async iterator를 반환하는 `makeAsyncIterator()` 메서드를 제공합니다.

`asyncDownloadIterator ` 를 활용해 bytes 를 하나씩 반복할 수 있습니다.

이제 모든 바이트를 수집하는 `accumulator` 를 추가합니다.

```swift
let accumulator = ByteAccumulator(name: name, size: size)

// 1번 루프 : 다운로드가 멈추지 않았는지, accumulator 가 bytes 를 더 수집할 수 있는지
while !stopDownloads, !accumulator.checkCompleted() {
	// 2번 루프 : 배치가 모두 차거나, sequence 가 완료될 때까지
  while !accumulator.isBatchCompleted,
        let byte = try await asyncDownloadIterator.next() {
    accumulator.append(byte)
  }
}
```

1번 루프를 먼저 살펴보면 2가지의 조건을 가지고 있습니다. 

이 두가지 조건을 활용하면 다운로드가 취소되지 않을 때 다운로드가 완료될 때까지 루프문을 유지할 수 있습니다. 
(취소는 이후에 다뤄보겠습니다.)

2번 루프 또한 2가지의 조건을 가지고 있습니다. 

accumulator 의 배치가 모두 차거나, sequence 가 완료될 때까지 accumulator 는 bytes 를 수집합니다.
