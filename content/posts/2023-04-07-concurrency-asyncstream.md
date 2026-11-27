+++
title = "[Concurrency] AsyncStream"
date = 2023-04-07T18:07:59+09:00
draft = false
tags = ["concurrency", "ios", "ios 13.0+", "swift"]
categories = ["iOS"]
tistory = "https://leetaek.tistory.com/74"
+++

![](/images/tistory/74/01.webp)

- `SwiftConcurrency`를 공부하다가 `AsyncStream`를 사용할만한 일이 생겨 적용하며 정리해보았다. 다음과 같이 인증 화면에서 시간을 카운트다운 하는 기능을 `AsyncStream`를 사용해 적용해보려고 한다.

# AsyncStream

- 비동기 Iterator를 제공하는 프로토콜인 [AsyncSequence](https://developer.apple.com/documentation/swift/asyncsequence)를 쉽게 구현할 수 있도록 제공되는 인터페이스

- `AsyncSequence`를 준수하기 때문에 `map`, `filter`, `contains` 등 익숙한 고차함수 메소드를 사용할 수 있다.

### 코드

```swift
@MainActor
func timerStream() -> AsyncStream<Int> {
  return AsyncStream<Int> { continuation in
    /// 시간 제한 5분
    var validTimeSeconds = 5 * 60 - 1

    Timer.scheduledTimer(withTimeInterval: 1.0, repeats: true){ timer in
      guard validTimeSeconds >= 0 else {
        timer.invalidate()
        continuation.finish()
        return
      }
      continuation.yield(validTimeSeconds)
      validTimeSeconds -= 1
    }
  }
}

/// 사용
Task {
  for await time in timerStream() {
    String(format: "%02d:%02d", time / 60, time % 60)
  }
}

/* 출력
	4:59
	4:58
  ...
	0:01
	0:00
*/
```

- `Async.Continuation`을 받는 클로저를 통해 초기화하여 데이터 스트림을 제공한다.

- `yield(_:)` 메소드를 동해 스트림에 데이터를 전달하고, `finish()` 메소드를 통해 스트림을 종료한다.

- `for await in `구문을 통해 사용한다.

# AsyncThrowingStream

- `AsyncStream`에서 에러 핸들링이 추가된 버전.

- `finish()`와 `yield(_:)` 외에 `finish(throwing: )` 추가되어 에러를 던질수 있다.

### 코드

```swift
@MainActor
func timerStream() -> AsyncThrowingStream<Int, Error> {
  return AsyncThrowingStream<Int, Error> { continuation in
    var validTimeSeconds = 5 * 60 - 1

    Timer.scheduledTimer(withTimeInterval: 1.0, repeats: true){ timer in
      guard self.validTimeSeconds >= 0 else {
        timer.invalidate()
        continuation.finish(throwing: TimerError.timeout)
        return
      }

      continuation.yield(validTimeSeconds)
      validTimeSeconds -= 1
    }
  }
}

/// 사용
Task {
  do {
    for try await seconds in timerStream() {
      self.validTimeLabel.text = String(format: "%02d:%02d", seconds / 60, seconds % 60)
    }
  }
  catch {
    if error as? TimerError == TimerError.timeout {
      /// 타임아웃 알럿 띄움
      await self.showExpiredTimeAlert()
    }
  }
}

/* 출력
	4:59
	4:58
  ...
	0:01
	0:00
	❗️타임아웃 알럿
*/
```

# AsyncStream vs Combine

- Concurrency 메소드들은 동시성 프로그램을 대응하기 위해 Swift 언어 자체에 내장된 오퍼레이션.

- Combine은 이벤트처리나 스트림 처리에 중점을 두고 설계를 한 프레임워크.

→ 분명 겹치는 부분이 있기 때문에 서로를 대체해서 사용할 수 있을 것 같다.

→ Combine을 가져다 쓰면 많은 작업을 쉽게 하지만 더 무거워질 것 같다. → SwiftUI에서는 Combine이 훨 편할듯 …?

# Reference

[AsyncStream | Apple Developer Documentation](https://developer.apple.com/documentation/swift/asyncstream/)

[[Swift 5.7+][Concurrency] AsyncStream, AsyncThrowingStream 알아보기 - Continuation vs unfolding](https://minsone.github.io/swift-concurrency-asyncstream-AsyncThrowingStream)

## 2026년에 다시 보며

Swift 5.9(SE-0388)부터 `AsyncStream.makeStream(of:)` 로 스트림과 continuation 을 한 번에 만들 수 있다. 이 글처럼 클로저 안에서 continuation 을 바깥 변수에 꺼내 두는 우회가 필요 없어진다.

```swift
let (stream, continuation) = AsyncStream.makeStream(of: Int.self)
continuation.yield(1)
continuation.finish()
```

스트림을 다 쓰면 `finish()` 를 부르고, 구독 쪽 Task 가 취소될 때 정리할 일이 있으면 `onTermination` 에 적어 두는 것은 그대로다.
