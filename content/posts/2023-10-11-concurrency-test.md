+++
title = "[Test] Concurrency Test"
date = 2023-10-11T10:21:13+09:00
draft = false
tags = ["concurrency", "ios", "swift", "test"]
categories = ["iOS"]
tistory = "https://leetaek.tistory.com/78"
+++

# 상황

`AsyncSequence`를 사용한 Concurrency 메서드를 테스트하는데 테스트가 종료되지 않는 상황이 발생했다. 로그도 뜨지 않고 왜 이런 상황이 발생하는 건지 파악도 잘 안됐다. 아마 테스트를 실행하는 스레드와 비동기 작업을 수행하는 스레드가 작업을 끝냈다는 것을 서로 전달하지 않아 데드락이 걸린게 아닐까 싶었다.

찾아보니 `XCTestExpectation`를 사용해서 비동기 메서드를 테스트할 때에 사용하는 코드가 있길래 이걸 조금 수정해서 사용하니 다행히 잘 돌아갔다.

# 해당 메서드 코드

```swift
import XCTest

extension XCTestCase {
  /// Test Concurrency method
  func execute(withTimeout timeout: TimeInterval,
               file: StaticString = #filePath,
               line: UInt = #line,
               workItem: @escaping (_ exp: XCTestExpectation) async throws -> Void) async throws {
    let expectation = XCTestExpectation(description: "wait for async function")
    let task = Task {
      do {
        try await workItem(expectation)
      }
      catch {
        XCTFail("\(error)", file: file, line: line)
      }
    }

    await fulfillment(of: [expectation], timeout: timeout)
    task.cancel()
  }

}
```

### XCTestExpectation

```swift
let expectation = XCTestExpectation(description: "wait for async function")
```

- 비동기 작업의 완료를 기다리고 테스트의 성공이나 실패를 판단하기 위해 사용하는 클래스.

- 해당 클래스의 `fulfill()` 메서드를 호출하면 기대값을 충족했다고 판단한다.

- 시간을 지정해서 지정된 시간 내에 기대값이 충족되지 않으면 테스트를 실패시킨다.

### Task

```swift
let task = Task {
  do {
    try await workItem(expectation)
  }
  catch {
    XCTFail("\(error)", file: file, line: line)
  }
}
```

- 파라미터로 받은 `async`메서드를 `Task`를 통해 비동기 작업을 수행한다.

- 해당 작업을 수행할 때에 파라미터로 위에서 선언한 `expectation`을 넘겨준다. 비동기 작업을 수행한 후 해당 작업이 완료되었음을 파라미터로 전달한 `expectation`를 통해 알린다.

### fulfillment

```swift
await fulfillment(of: [expectation], timeout: timeout)
```

- 기존에는 `wait(for:_, timeout:_)`메서드를 통해 `expectation`의 결과값을 기다리는 시간을 지정했었던 것 같다.

- 찾아보니 `fulfillment`메서드가 있어서 concurrency에 대응하게 수정했다.

- 작업이 끝나고` task.cancel()`로 태스크 종료

# 사용

```swift
func testSendThroughConcurrencyAdapter() async throws {
    /// 1. execute 메서드를 통해 테스트 수행,
    /// 5초의 대기시간 동안 기다린 후 fulfill이 호출되지 않으면 테스트 실패
    try await execute(withTimeout: 5) { exp in
      //given
      let adapter = NuguAdapter(textAgent: StubTextAgent())

      //when
      try await adapter.send(message: "안녕")
      for await response in adapter {
        if case let .success(message) = response {
          guard let text = try message?.text?.strippingHTML() else { return }
          // then
          XCTAssertEqual(text, "저를 잊지 않고 찾아줘서 기뻐요! 오늘도 웃음 가득한 하루 되길 바라요.")
          /// 2. 메서드 수행 후에 클로저를 통해 전달받은 expectation의 fulfill메서드 수행.
          exp.fulfill()
        }
      }
    }
  }
```

- 사용 흐름은 주석 참고!

# Reference

[fulfillment(of:timeout:enforceOrder:) | Apple Developer Documentation](https://developer.apple.com/documentation/xctest/xctestcase/4109476-fulfillment)

[Handling never finishing async functions in Swift package tests](https://augmentedcode.io/2022/10/31/handling-never-finishing-async-functions-in-swift-package-tests/)

## 2026년에 다시 보며

Xcode 16 부터는 XCTest 대신 Swift Testing 으로 같은 테스트를 쓸 수 있다. `@Test` 함수에 `async` 를 붙이면 바로 `await` 할 수 있고, 콜백이 정해진 횟수만큼 불렸는지는 `XCTestExpectation` 대신 `confirmation()` 으로 확인한다.

Swift 6 의 엄격한 동시성 검사를 켜면, 테스트에서 공유 상태를 건드리는 코드도 `Sendable` · 격리 규칙을 지켜야 컴파일된다. 이 글의 예제를 지금 옮긴다면 테스트 대상의 격리(`@MainActor` 등)부터 정해 두는 게 순서다.
