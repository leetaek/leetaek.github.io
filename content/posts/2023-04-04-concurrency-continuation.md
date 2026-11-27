+++
title = "[Concurrency]Continuation"
date = 2023-04-04T17:17:53+09:00
draft = false
tags = ["concurrency", "ios", "swift"]
categories = ["iOS"]
tistory = "https://leetaek.tistory.com/72"
+++

# Continuation

- Push를 등록을 설정하던 도중에 여러 콜백 함수로 인한 장풍이 생기고 뭔가 가독성이 거슬려서 전에 공부했던 `async/await`로 장풍 좀 없애고 싶다는 생각이 들었다.

- 요즘엔 많은 메소드들이 `async/await`로 구현되어서 편하게 구현하고 있어서 편하게 변환하고 있던 중 막히는 순간이 왔다.

```swift
/// 막혔던 예시
var pushTokenHandler: ((String?) -> Void)? = nil

func pushTokenRegister(handler: @escaping (_ fcmToken: String?) -> Void) {
	self.pushTokenHandler = handler
}

/// 사용
pushTokenRegister { fcmToken in
	print(fcmToken)
}
```

- 찾아보니 이런 비동기 코드들(`Callback`이나 `Delegate` 등)을 진행 상태를 async/await 함수와 같이 상황에 따라 **일시 중지(suspend)**와**재개(resume)**상태로 돌릴 수 있는 인터페이스를 제공한다. 이를 `Continuation`이라 한다.

**→ Continuation은 기존에 구현된 비동기 코드들을 wrapping하여 사용한다. →**위의 Thread 제어권에 대한 설명은 [데브시스터즈 기술 블로그](https://tech.devsisters.com/posts/crunchy-concurrency-swift/)에 매우매우 잘 설명이 되어있다!

## resume

- await로 suspend 된 지점에서 다시 작업을 재개하는 메소드

- 반환 값에 따라 `resume()`, `resume(returning: T)`, `resume(throwing: E)`, `resume(with: Result<T, E>)` 등을 제공한다.

- **❗️Continuation에서 resume 메소드는 반드시 단 한번 실행되야 한다.**

→ 실행되지 않으면 영원히 await, 2번 이상 실행되면 뭔 일이 일어날지 예상 못한다.

## CheckedContinuation

- **런타임에 resume 작업을 체크한다.**resume이 누락되거나 여러개 사용된 경우 명시적으로 Fatal error를 발생시킨다.

- 예외 처리에 따라 다음과 같은 메소드를 제공한다.

   - `withCheckedContinuation(function: _: )`

   - `withCheckedThrowingContinuation<T>(function:_:) `

## UnsafeContinuation

- **resume를 체크하지 않는다.**UnsafeContinuation은 event loops, delegate, callback을 낮은 오버헤드 매커니즘으로 처리하기 위해 만들어졌다.

- 예외 처리에 따라 다음과 같은 메소드를 제공한다.

   - `withUnsafeContinuation(_:)`

   - `withUnsafeThrowingContinuation<T>(_:) `

→ 두 메소드 모두 다른 변경 없이 서로 대체할 수 있다.

# 코드

```swift
var pushTokenHandler: ((String?) -> Void)? = nil

func pushTokenRegister(handler: @escaping (_ fcmToken: String?) -> Void) {
	self.pushTokenHandler = handler
}

/// 사용
pushTokenRegister { fcmToken in
	print(fcmToken)
}
```

```swift
/// pushTokenRegister
func pushTokenRegister() async -> String? {
	/// CheckedContinuation 사용
  return await withCheckedContinuation{ continuation in
		/// 기존의 pushTokenRegister
    pushTokenRegister { fcmToken in
			/// 해당 값을 continuation을 통해 resume
      continuation.resume(returning: fcmToken)
    }
  }
}

/// 사용
let fcmToken = await AppDelegate.shared.pushRegister()
```

- 해당 예에서는 resume(returning: T) 를 통해 값만 보냈지만 실제로는 resume(with: Result<T, E>) 를 통해 에러를 담아 보내는게 분기처리에 유리해 자주 쓰일듯하다.

# 결론

- 만들어보니 실제로 콜백으로 인한 들여쓰기가 줄어들고, 가독성도 편해진 것을 확인할 수 있었다.

- 성능 테스트는 안해봤는데 suspend와 resume로 컨텍스트 스위칭을 주려 성능이 조금 나아지지 않았을까 …? 사실 기존 메소드를 wrapping 한거라 확실하진 않는데 성능차이도 실제로 생길지 궁금하다.

# Reference

- Swift Concurrency

[Documentation](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/)

- checkedContinuation

[CheckedContinuation | Apple Developer Documentation](https://developer.apple.com/documentation/swift/checkedcontinuation)

- Thread 제어권

[바삭한 신입들의 동시성 이야기 - Swift편](https://tech.devsisters.com/posts/crunchy-concurrency-swift/)

## 2026년에 다시 보며

지금은 이 글에서 continuation 으로 감쌌던 Apple API 중 상당수가 `async` 버전을 직접 제공한다. 새로 감싸기 전에 문서에서 `async` 메서드가 있는지 먼저 보는 게 좋다.

감싸야 할 때는 `withCheckedContinuation` · `withCheckedThrowingContinuation` 을 쓰고, 모든 경로에서 `resume` 을 **정확히 한 번** 부르는 규칙은 변하지 않았다. Swift 6 에서는 넘기는 값이 `Sendable` 이어야 한다는 점도 함께 확인해야 한다.
