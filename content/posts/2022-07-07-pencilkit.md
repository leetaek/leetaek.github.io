+++
title = "[iOS] PencilKit"
date = 2022-07-07T17:22:33+09:00
draft = false
tags = ["ios", "swiftui", "pencilkit"]
categories = ["iOS"]
tistory = "https://leetaek.tistory.com/35"
+++

PencilKit은 2019년부터 애플에서 지원하는 프레임워크다. PencilKit은 터치와 Apple Pencil 입력을 그림으로 캡처하고 앱에 표시하는 기능을 지원한다. iOS13 이상부터 지원하며 앱에 그림 그리는 기능을 쉽게 구현할 수 있다.

#### PKCanvasView

```swift
class PKCanvasView : UIScrollView
```

 손가락이나 Apple Pencil을 통해 그려진 콘텐츠를 렌더링하여 앱에 표시하는 뷰. 그림을 그리고 표시하는 뷰라고 생각하면 될것 같다.  PKCanvasView에서 그려진 데이터는 PKDrawing에 저장된다.

#### PKDrawing

```swift
struct PKDrawing
```

PKCanvasView에서 그린 데이터를 저장하는 구조체. PKStroke형태로 저장하며 dataRepresentation이라는 메서드를 통해 Data형태로 반환받을 수 있다.

#### PKToolPicker

```swift
class PKToolPicker : NSObject
```

사용자가 팔레트와 같이 펜의 종류나 색상, 지우개나 자와 같은 도구를 선택할 수 있도록 제공되는 도구.

#### 사용

나의 경우 다음과 같이 ViewController를 만들고, SwiftUI로 사용하기 위해 UIViewControllerRepresentable로 가져와 사용했다.

![](/images/tistory/35/01.webp)

![](/images/tistory/35/02.webp)

PencilKit을 사용하려고 이런저런 코드들을 찾아보다 Github에서 참고하기 좋은 코드들을 찾았었다.

19년 WWDC에서 PencilKit 발표 당시의 sample code :

[https://github.com/legolasW/DrawingWithPencilKit](https://github.com/legolasW/DrawingWithPencilKit)

[GitHub - legolasW/DrawingWithPencilKit: willResignActiveNotification called multiple times. After you quit move move back and fo willResignActiveNotification called multiple times. After you quit move move back and fourth between collectionView and note. Then move the app to background, you'll observe willResignActiveNot... github.com](https://github.com/legolasW/DrawingWithPencilKit)

ViewController로 사용하고 SwiftUI에서 사용한 sample code  :

[https://github.com/nnovalbos/PencilKitDrawingInSwiftUI](https://github.com/nnovalbos/PencilKitDrawingInSwiftUI)

[GitHub - nnovalbos/PencilKitDrawingInSwiftUI: Ejemplo generado tras realizar curso de udemy -> PencilKit Drawing in SwiftUI for Ejemplo generado tras realizar curso de udemy -> PencilKit Drawing in SwiftUI for iOS and iPadOS - GitHub - nnovalbos/PencilKitDrawingInSwiftUI: Ejemplo generado tras realizar curso de udemy -&g... github.com](https://github.com/nnovalbos/PencilKitDrawingInSwiftUI)

PencilKit 개발자 공식 문서 :

[https://developer.apple.com/documentation/pencilkit](https://developer.apple.com/documentation/pencilkit)

[Apple Developer Documentation   developer.apple.com](https://developer.apple.com/documentation/pencilkit)

## 2026년에 다시 보며

이 글을 쓸 때 만지작거리던 PencilKit 으로, 지금은 iPad 성경 필사 앱 [새기다](/carve/)를 만들고 있다. 그 뒤의 이야기 — TCA 로 앱을 구성하고 AI 와 함께 개발한 기록 — 는 [코드를 덜 읽는 대신 무엇을 확인했나](/posts/2026-11-09-ai-orchestration-06-verification/)에 이어서 적었다.
