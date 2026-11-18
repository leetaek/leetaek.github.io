+++
title = "[Tuist] 4.x로 업데이트하며 느낀 경험 (feat. TCA)"
date = 2024-03-15T11:52:53+09:00
draft = false
tags = ["firebase", "ios", "tca", "tuist", "xcode"]
categories = ["iOS"]
tistory = "https://leetaek.tistory.com/80"
+++

# 업데이트 계기

원래 사용하던 Tuist의 버전은 3.x버전이었다. 4.x대로 업데이트 할 생각은 없었다.

개인적으로 혼자 사이드 프로젝트로 진행하던 아이패드용 앱이 있는데, SwiftUI + Composable Architecture(이하 TCA)를 사용하며 Tuist로 관리하는 프로젝트였다. TCA의 빠른 업데이트 속도를 따라가는것도 벅차서 Tuist 까지 업데이트를 따라가기엔 부담이 있었기 때문이었다.

그러던 어느날, 이런 에러가 출력되었다.

![](/images/tistory/80/01.png)

해당 에러는 Xcode 15.3으로 업데이트 되면서 올라가면서 Xcodebuild의 버전도 같이 올라가면서 생긴 문제인것 같다. Xcodebuild의 버전을 15.2으로 사용하니 원래대로 빌드가 되었다.

이왕이면 업데이트 된 Xcode 15.3버전에서 작업을 하고 싶었고, Swift5.10도 한번 둘러보고 싶었다. 그래서 Tuist 를 업데이트 하면 문제가 해결될 수도 있지 않을까 하는 생각으로 현재 가장 최신 버전인 Tuist 4.6.0 버전으로 업데이트하게 되었다.

Tuist를 업데이트 해도 Xcodebuild 15.3에서는 위의 문제가 뜨는데 혹시나 해결법을 아신다면 댓글 부탁드립니다 ㅠㅡㅠ

# 변경점

아래는 업데이트하고 직접 사용해 본 변경점들이다.

### Dependencies

![](/images/tistory/80/02.png)

Tuist의 SPM을 사용하는 경우 기존에는 Dependencies.swift 에 Dependencies의 swiftPackageManager로 사용하려는 Package들을 넘겨줘야 했다. 4.x로 업데이트 되면서 Dependencies.Swift가 Package.swift로 변경되었다.

```swift
/// Tuist3.x 버전
/// Dependencies.swift
import ProjectDescription
import ProjectEnvironment

let dependencies = Dependencies(
    swiftPackageManager: .packages,
  platforms: [.iOS]
)

/// Plugins/"MyApp"/ProjectDescriptionHelpers/Library.swift
extension SwiftPackageManagerDependencies {
    public static var packages: Self = .init(
        [
            .package(url: "<https://github.com/firebase/firebase-ios-sdk.git>", from: "10.15.0"),
            .package(url: "<https://github.com/onevcat/Kingfisher.git>", from:  "7.9.1"),
            .package(url: "<https://github.com/realm/realm-swift>", from: "10.47.0")
        ],
        productTypes: [
            "Alamofire": .framework,     // dynamicFramework
        ],
        targetSettings: [
        ...
        ]
    )
}
```

```swift
/// Tuist4.x 버전
import PackageDescription

#if TUIST
import ProjectDescription
import ProjectDescriptionHelpers

let packageSettings =  PackageSettings(
        productTypes: [
            "Alamofire": .framework, // dynamicFramework
        ]
    )
#endif

let packages = Package(
    name: "PackageName",
    dependencies: [
        .package(url: "<https://github.com/Alamofire/Alamofire>", from:  "5.8.1"),
        .package(url: "<https://github.com/onevcat/Kingfisher.git>", from:  "7.9.1"),
        .package(url: "<https://github.com/firebase/firebase-ios-sdk.git>", from: "10.15.0")
    ]
)
```

### CLI

![](/images/tistory/80/03.png)

Tuist fetch 가 Tuist install로 변경되었다.

### Cache

예전부터 있었던 기능인거 같은데 외부 의존성을 미리 빌드해 캐시를 생성해둘수 있다고 한다. tuist cache warm 에서 tuist cache로 바뀌었다. 자세한 내용은 Document 참조.
 <https://docs.tuist.io/documentation/tuist/binary-caching>

- [Documentation](https://docs.tuist.io/documentation/tuist/binary-caching) — docs.tuist.io

### Macro

이전에 Tuist에는 Swift Macro 관련 기능에 버그가 있었다. 문제는 TCA가 1.4버전부터 Reducer를 Swift Macro를 처리하기 시작해서 Tuist의 SPM으로 가져오면 해당 의존성을 찾지 못하는 에러가 발생했었다.

![](/images/tistory/80/04.png)

해결법을 찾는데 종속성을 직접 추가해야 하는 과정이 필요하다고 해서 Tuist +SPM 대신 일단 Xcode의 SPM으로 추가해서 사용하고 있었다.

그런데 아래에 4.x버전으로 올리며 겪은 문제를 해결하는 과정에서 TCA를 Tuist의 SPM으로 추가해보니 잘 동작했다. 찾아보니 Tuist에서 해당 문제를 수정했다고 한다. 오예!

[Add support for transitive Swift Macros by pepicrft · Pull Request #5772 · tuist/tuist](https://github.com/tuist/tuist/pull/5772)

- [Add support for transitive Swift Macros by pepicrft · Pull Request #5772 · tuist/tuist](https://github.com/tuist/tuist/pull/5772) — github.com

# 4.x버전으로 올리며 겪은 문제

### duplicated symbols

![](/images/tistory/80/05.png)

![](/images/tistory/80/06.png)

*3.x버전에서 사용하던 Dependencies*

기존에는 TCA는 Macro, Realm은 ObjC 파일과 관련된 이슈가 있어서 Xcode의 SPM을 사용했었다. 그런데 동일한 구성으로 Tuist 4.x 로 업데이트하니 Firebase 쪽에서 duplicated symbols 란 에러를 뿜어냈다. Build Settings의 Other Linker Flags 설정과 관련된 에러라고 찾아볼 수 있었다.

all_load, -Xlinker, -no_warn_duplicate_libraries 등 다양하게 시도를 해봤으나 문제는 동일했다. -ObjC 를 제거하면 빌드는 되는데 사용하는 라이브러리들에 Objective-C가 있기 때문에 런타임에 문제를 일으킬것 같아서 다른 방법을 찾아보기로 했다. 시도하면서 알게된건 -ObjC 플래그는 Xcode가 자동으로 추가하는 것 같다.

Friebase의 의존성 설정을 잘못했나 싶어 [Tuist Sample에서 Package 설정](https://github.com/tuist/tuist/tree/63c8061aca43536d7b746db07dec5eb29838ab6a/fixtures/multiplatform_app_with_sdk)도 다시 확인해봤지만 다른점은 없다고 생각했다.

오류로 뿜어낸 Duplicated라는 단어에 혹시 Tuist+SPM과 Xcode+SPM 의 중복된 코드가 충돌이 나는건가 싶었다. 그래서 ObjC를 사용하는 Realm과 Firebase를 동일하게 몰아봤다.

문제가 되는 Firebase를 Xcode+SPM으로 가져왔을 때에는 문제가 발생하지 않았다. 내가 Tuist에서 설정을 잘못했나 싶었는데, 반대로 Realm을 Tuist+SPM 으로 설정했을 때에도 프로젝트가 성공적으로 빌드가 됐다. 예전에 동일한 방법으로 Realm을 추가할 땐 Objc 헤더를 못찾아서 빌드조차 안됐는데 왜…????

일단 내가 사용하고 있던 기능들은 문제가 없이 돌아가는 것 같다. Tuist 릴리즈 노트를 볼때에 ObjC 관련 업데이트가 지속적으로 있었던 것 같기는 한데 어떤 업데이트로 인해 가져올 수 있었는지 아직 파악하지 못했다. 구랴소 라이브러리의 모든 기능들이 모두 정상동작 하는지는 테스트를 더 해봐야할 것 같은데 혹시나 사용하다가 문제가 생긴 부분이 발견되면 호다닥 돌아와서 포스트 수정하겠습니다!

덤으로 Macro 때문에 Xcode+SPM으로 추가했던 Tuist로 추가하니 TCA도 정상적으로 돌아간다. 얘는 1.8버전에서 몇몇 클래스를 못찾는다고 오류가 떴는데, 최신 버전으로 업데이트하니 정상동작한다. 오예!

# 결론

- 뭐든 업데이트는 신중히 → 사용하는 주요 라이브러리들의 업데이트를 잘 확인하자
- 참고로 나는 블로그를 노션에서 작성해서 N2T로 올렸는데,,,, 24년 2월부터 Tistory가 API 지원을 중단해서 다 작성한걸 다시 블로그에 옮겨 적었다....  업데이트와 지원을 꾸준히 확인하자 ㅠㅠ

### Study Todo List

요 일을 통해 필요를 느낀 스터디 리스트

- Macro
- Linker 관련 설정
- Tuist 환경에서 CI/CD

# Reference

[GitHub - tuist/tuist: 🚀 Create, maintain, and interact with Xcode projects at scale](https://github.com/tuist/tuist)

- [GitHub - tuist/tuist: 🚀 Create, maintain, and interact with Xcode projects at scale](https://github.com/tuist/tuist) — github.com

[Documentation](https://docs.tuist.io/documentation/tuist/installation)

- [Documentation](https://docs.tuist.io/documentation/tuist/installation) — docs.tuist.io

[iOS Framework의 duplicate symbol 에러 해결하기](http://theeye.pe.kr/archives/2465)

- [iOS Framework의 duplicate symbol 에러 해결하기 | 아이군의 블로그](http://theeye.pe.kr/archives/2465) — theeye.pe.kr
