+++
title = "[UIKit] preferredMaxLayoutWidth, IntrinsicContentSize, Priority"
date = 2023-07-03T14:44:26+09:00
draft = false
tags = ["ios", "ui", "uikit"]
categories = ["iOS"]
tistory = "https://leetaek.tistory.com/77"
+++

# preferredMaxLayoutWidth

- 여러 줄의 `UILabel`에 적용될 수 있는 프로퍼티. `UILabel`의 **최대 너비**를 설정한다. 최대 너비를 넘어가는 경우 `UILabel`은 새로운 줄로 개행하며 높이를 증가시킨다.

- `preferredMaxLayoutWidth`를 설정해서 일정한 텍스트 레이아웃을 정렬하거나 적절한 레이아웃을 유지할 수 있다.

- Storyboard에서는 Desired Width에서 설정할 수 있다 . 이미지에서는 `preferredMaxLayoutWidth`를 설정했지만 Lines = 1인 상태이기 때문에 멀티라인이 아니므로 아무런 변화가 없다.

- Lines = 0으로 했을 때에 `preferredMaxLayoutWidth`가 적용되어 너비에 맞게 개행된걸 확인할 수 있다.

- 나는 Custom Label을 만들기 위해 코드로 다음과 같이 설정해 사용했다. 다음 코드에서는 text가 입력되어 bounds가 설정될 때에 최대 너비를 width에서 padding을 뺀 값으로 설정한다.

```swift
class CustomLabel: UILabel {
  // ... 코드들
  override var bounds: CGRect {
    didSet {
      preferredMaxLayoutWidth = bounds.width - (leftInset + rightInset)
    }
  }
}
```

# IntrinsicContentSize

- View의 내부 콘텐츠를 자연스럽게 표시하기 위한 최소한의 크기. View에 자체적으로 `IntrinsicContentSize`를 가지는 View들도, 없는 View들도 있다.

- 대부분의 View들이 `IntrinsicContentSize`를 가지고 있지만, **UIView는 IntrinsicContentSize가 없다.** View의 위치만 잡았을 때에 `UILabel`은 `IntrinsicContentSize`가 있기 때문에 괜찮지만 `UIView`는 크기를 잡아달라고 에러를 띄운다.

- `IntrinsicContentSize`의 유무는 view.IntrinsicContentSize 프로퍼티를 찍어서 확인할 수 있다. 해당 값이 -1이라면 `IntrinsicContentSize`가 없는 것이다.

```swift
let view = UIView()

print(view.view.intrinsicContentSize)

// 출력
// (-1, -1)
```

- 물론 `UIView`의 경우에도 원한다면 `IntrinsicContentSize`를 적용할 수 있다.

```swift
class CustomView: UIView {
  let btn: UIButton = {
    // 코드
  }()

  override var intrinsicContentSize: CGSize {
    let height = btn.frame.size.height
    let width =  btn.frame.origin.x + btn.frame.size.width
    return CGSize(width: width, height: height)
  }
}
```

- `IntrinsicContentSize`가 있는 View들은 AutoLayout 적용 시에 우선순위 문제가 발생한다. 해당 문제는 **`Priority`를 조정하여 해결할 수 있다.**

## invalidateIntrinsicContentSize()

- `IntrinsicContentSize`를 무효화한다. 즉 view의 크기에 변화가 있을 때에 `invalidateIntrinsicContentSize()` 메소드를 호출하여 기존의 `IntrinsicContentSize`를 무효화하고, 새로운 `IntrinsicContentSize`를 계산하여 변화된 크기를 적용한다.

# Priority

### Content Hugging Priority

- `IntrinsicContentSize`보다 커지는 것을 허용하지 않는다. → 우선순위가 낮은 뷰의 크기가 늘어난다.

- 기본값은 250이며, 다음과 같이 코드를 통해서도 설정할 수 있다.

```swift
let spacer: UILabel = {
  let view = UILabel()
  // 가로 방향으로 고유 컨텐츠보다 커지는걸 절대 허용하지 않음.
  view.setContentHuggingPriority(.required, for: .horizontal)
  // 세로 방향으로 우선순위에 따라 고유컨텐츠보다 늘어날 수 있음.
  view.setContentHuggingPriority(.defaultLow, for: .vertical)
  return view
}()
```

### Content Compression Resistance Priority

- `IntrinsicContentSize`보다 작아지는 것을 허용하지 않는다. → 우선순위가 낮은 뷰의 크기가 작아진다.

- 기본값은 750이며, 다음과 같이 코드를 통해서도 설정할 수 있다.

```swift
let spacer: UILabel = {
  let view = UILabel()
  // 가로 방향으로 고유 컨텐츠보다 줄어드는걸 절대 허용하지 않음.
  view.setContentCompressionResistancePriority(.required, for: .horizontal)
  // 세로 방향으로 우선순위에 따라 고유컨텐츠보다 줄어들 수 있음.
  view.setContentCompressionResistancePriority(.defaultLow, for: .vertical)
  return view
}()
```

# Reference

[invalidateIntrinsicContentSize() | Apple Developer Documentation](https://developer.apple.com/documentation/uikit/uiview/1622457-invalidateintrinsiccontentsize/)

[[iOS] preferredMaxLayoutWidth, Intrinsic content size, Content hugging priority, content Compression resistance priority](https://beankhan.tistory.com/41)

## 2026년에 다시 보며

SwiftUI 로 넘어오면서 같은 고민을 다른 이름으로 다시 만난다. Content Hugging · Compression Resistance 의 역할은 `.fixedSize()` 와 `.layoutPriority(_:)` 가 비슷하게 맡는다. UIKit 뷰를 `UIViewRepresentable` 로 감쌀 때 크기가 이상하게 잡히면, 이 글에서 본 `intrinsicContentSize` 와 우선순위 설정을 먼저 의심해 보면 된다.
