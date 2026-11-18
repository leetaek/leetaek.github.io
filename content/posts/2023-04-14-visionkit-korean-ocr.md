+++
title = "[SwiftUI] VisionKit을 통해 카메라로 찍은 이미지에서 한글 OCR 구현하기"
date = 2023-04-14T13:56:14+09:00
draft = false
tags = ["ios", "swift", "swiftui", "ui"]
categories = ["SwiftUI"]
tistory = "https://leetaek.tistory.com/76"
+++

- iOS 11부터 애플이 지원하는 [Vision](https://developer.apple.com/documentation/vision) 이라는 프레임워크가 있다. 머신러닝 기능을 활용하여 이미지나 비디오, 얼굴이나 바코드, 텍스트 등을 인식하여 다양한 작업을 수행할 수 있도록 하는 프레임워크다.

- 애플에서 이용해 직접 지원하다보니 빠르고 효율적이며, **무료이다…!**

- 이를 활용하여 이미지에서 텍스트를 인식하는 OCR 기능을 사용할 수 있는데, `iOS 16.0`부터 드디어 지원 언어에 한국어도 포함이 되었다. 이를 통해 간단하게 이미지로부터 한국어 텍스트를 인식하는 기능을 만들어 보려고 한다.

- 아래는 `iOS 16.4`에서 지원 언어와 `iOS 15.5`에서 Vision의 지원 언어이다. 한국어(ko-KR) 가 추가되어 있다.

  ![](/images/tistory/76/01.webp)

  ![](/images/tistory/76/02.webp)

# 준비 → 카메라 및 앨범

- 사실 이미지를 불러와도 됐지만 직접 찍은 사진을 그때그때 앨범에서 사진을 가져오는 기능을 구현해보고 싶었다. Enebin 님이 [포스트로 굉장히 잘 정리해두신걸](https://enebin.medium.com/list/swiftui-featmvvm-c928b54e9408) 활용했다. 해당 글들 중 2번까지만 구현해도 무방하다.

# ViewModel

- OCRViewModel은 간단하다. `Vision` 을 통해 읽은 텍스트를 받을 프로퍼티와 이미지 분석을 request를 요청할 메소드 하나면 된다.

```swift
import Vision
import VisionKit

class OCRViewModel: ObservableObject {
  // MARK: - Properties
  @Published var OCRString: String?

  // MARK: - Methods
  func recognaizeText(image: UIImage) {
		 ...
  }
}
```

### VNImageRequestHandler

- `Vision`은 `VNImageRequestHandler`를 통해 요청을 전달하고 처리한다. `VNImageRequestHandler`는 CGImage, data, url 등 다양한 파라미터를 받아 이미지를 처리하는데 요런건 개발자문서에서 필요한걸 확인하자.

  ```swift
  func recognaizeText(image: UIImage) {
    guard let Image = image.cgImage else { fatalError("이미지 오류")}

    let handler = VNImageRequestHandler(cgImage: Image, options: [:])
  }
  ```

- 기본적으로 이미지로 수직으로 받는 향한다고 가정(아래의 오른쪽 이미지)하고 처리한다. 이미지 방향이 수직이 아닌 경우 init에 orientation 옵션을 줄 수 있는데, 옵션의 방향의 경우 해당 UIImage의 `imageOrientation`프로퍼티로 이미지 방향을 확인할 수 있다. 그냥 사용하면 안되고 `CGImagePropertyOrientation`로 변환하여 사용해야 하는데 방법은 [CGImagePropertyOrientation의 개발자 문서](https://developer.apple.com/documentation/imageio/cgimagepropertyorientation)에 잘 나와있다.

![](/images/tistory/76/03.webp)

### VNRecognizeTextRequest

- 이미지에서 텍스트를 찾고 구성할 수 있도록 분석하는 request. Text 외에도 비전에는 Face, Body, Animal, Object 등을 분석할 수 있는 request가 있다.

- 해당 결과는 `VNRecognizedTextObservation` 객체로 반환되며 클로저를 통해 해당 값을 처리할 수 있다.

  ```swift
  let request = VNRecognizeTextRequest {  [weak self] request, error in
    // 결과값 옵셔널 바인딩
    guard let result = request.results as? [VNRecognizedTextObservation],
          error == nil else { return }
    // resultd에서 최대 후보군 1개에서 첫번째 후보를 String 타입으로 변환
    let text = result.compactMap { $0.topCandidates(1).first?.string }
  		.joined(separator: "\n")
    // 이를 OCR 결과에 반영
    self?.OCRString = text
  }
  ```

- 그 외에도 request와 관련된 설정들을 추가할 수 있다.

  ```swift
  /// 언어를 인식하는 우선순위 설정
  if #available(iOS 16.0, *) {
    /// 앱의 지원 버전에 따라 가장 최신 revision을 default로 지원해주기 떄문에
    /// 사실 안해도 상관 없다...
    /// VNRecognizeTextRequestRevision3는 iOS 16부터 지원
    request.revision = VNRecognizeTextRequestRevision3
    request.recognitionLanguages = ["ko-KR"]
  } else {
    request.recognitionLanguages = ["en-US"]
  }
  /// 정확도와 속도 중 어느 것을 중점적으로 처리할 것인지
  request.recognitionLevel = .accurate
  /// 언어를 인식하고 수정하는 과정을 거침.
  request.usesLanguageCorrection = true
  ```

### request 실행

- 구성한 `VNImageRequestHandler`를 통해 `VNRecognizeTextRequest`를 요청하면 된다.

  ```swift
  do {
    print(try request.supportedRecognitionLanguages())
    try handler.perform([request])
  } catch {
    print(error)
  }
  ```

# View

### Image Picker

- ImagePicker의 프로퍼티로 선택한 이미지를 OCR View로 넘겨줄 image와 ImagePicker 뷰를 띄울지 결하는 isPresent를 선언했다.

  ```swift
  struct ImagePicker {
    @Binding var image: UIImage?
    @Binding var isPresented: Bool
  }
  ```

- 카메라로 찍은 사진을 선택해 가져와야하는데, SwiftUI에서 ImagePicker를 지원하지 않기 때문에 UIKit의 `UIImagePickerController`를 `UIViewControllerRepresentable`로 가져와 사용했다.

- `UIViewControllerRepresentable`에서 `Coordinator`로 delegate 를 처리한다. 이미지를 선택하면 선택한 이미지를 image 프로퍼티에 할당하고, ImagePicker 뷰를 닫는다.

  ```swift
  extension ImagePicker: UIViewControllerRepresentable {
    typealias UIViewControllerType = UIViewController

    func makeUIViewController(context: Context) -> UIViewController {
      let picker = UIImagePickerController()
      picker.delegate = context.coordinator
      return picker
    }

    func updateUIViewController(_ uiViewController: UIViewController, context: Context) {
    }

    func makeCoordinator() -> Coordinator {
      return Coordinator(self)
    }

    class Coordinator: NSObject, UINavigationControllerDelegate, UIImagePickerControllerDelegate {
      let parent: ImagePicker

      init(_ parent: ImagePicker) {
        self.parent = parent
      }

      func imagePickerController(_ picker: UIImagePickerController, didFinishPickingMediaWithInfo info: [UIImagePickerController.InfoKey : Any]) {
        guard let image = info[.originalImage] as? UIImage else { return }
        self.parent.image = image
        self.parent.isPresented = false
      }
    }
  }
  ```

- CameraViewModel에 ImagePicker에 바인딩할 image와 present 프로퍼티를 추가한다.

  ```swift
  class CameraViewModel: ObservableObject {
    // MARK: - Properties
    ...
    @Published var selectedImage: UIImage?
    @Published var imagePickerPresented: Bool = false
  }
  ```

> 찾아보니 iOS 16.0 와 Xcode 14.0 부터 [PhotosPicker](https://developer.apple.com/documentation/photokit/photospicker/)라는 뷰를 지원한다. 어차피 Vision에서 한국어 언어를 지원하는 것도 16.0 이상이니 추후에 PhotosPicker를 사용하는 방법으로 수정해봐야겠다.

### OCRView

- 이미지와 Vision을 통해 인식한 텍스트를 출력할 뷰를 만든다. 레이아웃은 입맛에 맞게 구성하면 된다.

  ```swift
  import SwiftUI

  struct OCRView: View {
    // MARK: - Properties
    @ObservedObject private var viewModel = OCRViewModel()
    var image: UIImage

    // MARK: - Views
    var body: some View {
      imageView
          .onAppear {
            viewModel.recognaizeText(image: image)
          }
    }

    var imageView: some View {
      VStack() {
        Image(uiImage: image)
          .resizable()
          .scaledToFit()

      ScrollView {
        Text(viewModel.OCRString ?? "")
          .lineLimit(nil)
          .multilineTextAlignment(.leading)
      }
    }
  }
  ```

- CameraViewModel에 OCRView Present 여부를 결정하는 프로퍼티를 추가한다.

  ```swift
  class CameraViewModel: ObservableObject {
    // MARK: - Properties
    ...
    @Published var selectedImage: UIImage?
    @Published var imagePickerPresented: Bool = false
    @Published var OCRViewPresented: Bool = false
  }
  ```

### CameraView

- CameraView에 ImagePicker와 OCRView를 추가한다. 나는 `sheet`로 붙혔다.

- ImagePicker의 sheet에서 `onDismiss`에서 OCRViewPresented를 바꿔 ImagePicker가 닫히며 OCRView가 뜨도록 설정한다. 이때 ImagePicker에서 선택한 Image를 viewModel을 통해 OCRView로 넘겨준다.

  ```swift
  struct CameraView: View {
    // MARK: - Properties
    @ObservedObject var viewModel = CameraViewModel()

    // MARK: - Views
    var body: some View {
      /// 기존에 Enebin이 구현하신 기존의 뷰
      /// 버튼들만 뷰로 따로 나눴다.
      ZStack {
        viewModel.cameraPreview.ignoresSafeArea()
          .onAppear {
            viewModel.config()
          }

        VStack {
          upSideButtons
          Spacer()
          downSideButtons
        }
      }
      /// ImagePicker
      .sheet(isPresented: $viewModel.imagePickerPresented, onDismiss: {
          viewModel.OCRViewPresented.toggle()
        }) {
          ImagePicker(image: $viewModel.selectedImage, isPresented: $viewModel.imagePickerPresented)
        }
      /// OCRView
      .sheet(isPresented: $viewModel.OCRViewPresented) {
        if let image = viewModel.selectedImage {
          OCRView(image: image)
        }
      }
    }
  }
  ```

- ImagePicker를 띄울수 있도록 하단 왼쪽 앨범 버튼 액션에 toggle을 추가한다.

  ```swift
  /// 앨범 버튼
  Button {
    viewModel.imagePickerPresented.toggle()
  } label: {
    if let previewImage = viewModel.recentImage {
      Image(uiImage: previewImage)
        .resizable()
        .scaledToFill()
        .frame(width: 50, height: 50)
        .clipShape(RoundedRectangle(cornerRadius: 10))
        .aspectRatio(1, contentMode: .fit)
        .padding()

    } else {
      RoundedRectangle(cornerRadius: 10)
        .stroke(lineWidth: 2)
        .frame(width: 50, height: 50)
        .padding()
    }
  }
  ```

# 결과

![](/images/tistory/76/04.webp)

![](/images/tistory/76/05.webp)

- 매우 빠르게 잘 인식한다. 오른쪽에 저런 정신없는 이미지도 나름 인식한다.

![](/images/tistory/76/06.webp)

![](/images/tistory/76/07.webp)

![](/images/tistory/76/08.webp)

- 삐둘빼뚤한 손글씨도 인식하기는 하는데, 세로쓰기는 버거워하는 것 같다. 최고심같이 화려한 이미지도 `Vision`이 많이 아파하는 것 같다.

- 영수증도 인식을 해봣는데 선명한 영수증은 잘 인식은 하는데, 흐릿한 영수증들은 잘 안읽히기도 했다. 그리고 영수증이 띄어진 글자들이 많고 세로로 길다보니 인식할 텍스트를 어떻게 처리할지 고민이 필요한 것 같다.

# Reference

- [Detecting Objects in Still Images | Apple Developer Documentation](https://developer.apple.com/documentation/vision/detecting_objects_in_still_images)

- [List: SwiftUI만 써서 호다닥 카메라앱 만들기 feat.MVVM | Curated by Enebin | Medium](https://enebin.medium.com/list/swiftui-featmvvm-c928b54e9408)

- [Swift ) 한글 스캔 OCR 시도한 것들 정리 - EEYatHo iOS](https://eeyatho.tistory.com/140)

- [[iOS/Swift] VisionKit OCR Api 예제 한글](https://ohwhatisthis.tistory.com/17)

Uploaded by [N2T](https://github.com/jmjeon94/N2T)
