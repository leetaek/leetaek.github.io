+++
title = "SwiftUI 스크롤이 버벅인 이유는 현지화 문자열이었다"
date = 2026-11-27T11:45:39+09:00
draft = false
description = "SwiftUI 목록의 줄마다 현지화 문자열을 조회하다가 스크롤이 버벅였고, 언어마다 한 번만 만들게 바꿔 62~70fps 로 돌린 기록입니다."
tags = ["ios", "swift", "swiftui", "performance", "localization", "새기다"]
categories = ["iOS"]
+++

새기다 3.0 에서 iPhone 용 읽기 화면을 만들다가, 긴 장을 스크롤할 때 화면이 눈에 띄게 버벅이는 걸 발견했습니다. 원인은 렌더링이 아니라 현지화 문자열 조회였습니다. 어떻게 찾았고 어떻게 고쳤는지 정리합니다.

## 문제

읽기 화면은 한 장을 세로로 스크롤하며 읽는 화면입니다. 절마다 줄이 하나씩 있고(`LazyVStack`), 줄마다 「크게 보기 · 이미지 저장 · 공유 · 즐겨찾기」 컨텍스트 메뉴가 붙어 있습니다.

![절 줄을 길게 눌러 띄운 컨텍스트 메뉴](/images/localized-string-per-row-fps/localized-string-per-row-fps-1-context-menu.png)


iPhone 16e · iOS 26.2 에서 176절짜리 시편 119편을 스크롤하면 평균 약 38fps 가 나왔습니다. 짧은 장에서는 티가 덜 났지만, 긴 장에서는 손가락을 따라오지 못하는 게 바로 느껴졌습니다.

## 원인

### 먼저 의심한 것은 원인이 아니었습니다

직전 수정판에서 줄에 `compositingGroup` 을 넣은 적이 있어서 그것부터 의심했습니다. 그래서 그 수정이 들어가기 전 판과 들어간 판을 같은 장, 같은 조작으로 번갈아 재 봤습니다. 둘 다 똑같이 버벅였습니다. 렌더링 쪽 변경은 원인이 아니었습니다.

### sample 로 메인 스레드를 봤습니다

스크롤하는 동안 macOS 의 `sample` 로 시뮬레이터 앱 프로세스의 메인 스레드를 떴습니다.

```bash
# 스크롤하는 동안 5초간 메인 스레드 표본을 뜹니다
sample <앱 프로세스 이름> 5 -file ~/Desktop/scroll.sample.txt
```

메인 스레드가 바쁜 표본의 약 40% 가 `_copyStringTable` 아래에서 파일을 열고(open) 읽고(read) 있었습니다. 현지화 문자열 표, 즉 `.strings` 를 읽는 코드입니다.

![sample 결과 일부 — 절 줄을 그리는 코드 아래에서 _copyStringTable 이 파일을 열고 읽습니다](/images/localized-string-per-row-fps/localized-string-per-row-fps-2-sample.png)

위 그림은 글을 쓰며 수정 전 코드로 되돌려 다시 뜬 `sample` 결과에서 해당 부분만 줄여 옮긴 것입니다. 앱 쪽 이름은 예제 코드에 맞춰 바꿨습니다.


### 줄을 그릴 때마다 문구 넷을 새로 만들고 있었습니다

새기다는 String Catalog(`Localizable.xcstrings`)로 현지화합니다. 한 파일에 키와 언어별 번역을 모아 두면, 빌드할 때 언어마다 `.strings` 파일로 바뀌어 앱 번들에 들어갑니다. 위 표본에서 읽고 있던 게 이 파일입니다.

![String Catalog — 키 하나에 언어별 번역이 모여 있습니다](/images/localized-string-per-row-fps/localized-string-per-row-fps-3-string-catalog.png)

또 새기다는 앱 안에서 언어를 바로 바꾸는 기능이 있습니다. 그래서 문구를 꺼낼 때 시스템 언어 대신 앱이 고른 locale 을 넘겨 풀어 주는 함수를 씁니다. 대략 이런 모양입니다(설명용으로 새로 쓴 코드입니다).

```swift
enum L10n {
  static func string(_ value: String.LocalizationValue, locale: Locale) -> String {
    let resource = LocalizedStringResource(
      value,
      locale: locale,
      bundle: .atURL(Bundle.module.bundleURL)
    )
    return String(localized: resource)
  }
}
```

그리고 절 줄의 메뉴 항목을 줄을 그릴 때마다 만들고 있었습니다.

```swift
// 줄을 그릴 때마다 불림 → 한 줄에 문구 4개
func menuItems(isFavorite: Bool, locale: Locale) -> [MenuItem] {
  [
    MenuItem(title: L10n.string("크게 보기", locale: locale)),
    MenuItem(title: L10n.string("이미지 저장", locale: locale)),
    MenuItem(title: L10n.string("공유", locale: locale)),
    MenuItem(title: isFavorite
      ? L10n.string("즐겨찾기 해제", locale: locale)
      : L10n.string("즐겨찾기에 추가", locale: locale)),
  ]
}
```

표본으로 보면 이 방식은 **부를 때마다 문자열 표 파일을 다시 열었습니다.** 스크롤하면 줄이 계속 새로 그려지니, 줄마다 파일 I/O 가 네 번씩 메인 스레드에서 일어난 셈입니다.

모든 `String(localized:)` 가 매번 파일을 연다는 뜻은 아닙니다. 확인한 건 「bundle 을 URL 로 넘기는 `LocalizedStringResource` 를 이렇게 자주 부를 때 문자열 표를 다시 읽었다」까지입니다.

## 개선

문구는 언어가 같으면 항상 같습니다. 그래서 **언어마다 한 번만** 만들어 두고 다시 쓰게 했습니다.

```swift
@MainActor
final class MenuCopy {
  struct ItemSets {
    var plain: [MenuItem]
    var favorite: [MenuItem]
  }

  private var cached: (localeID: String, sets: ItemSets)?

  func itemSets(locale: Locale) -> ItemSets {
    if let cached, cached.localeID == locale.identifier {
      return cached.sets
    }
    let sets = ItemSets(
      plain: menuItems(isFavorite: false, locale: locale),
      favorite: menuItems(isFavorite: true, locale: locale)
    )
    cached = (locale.identifier, sets)
    return sets
  }
}

// 화면이 하나 들고 있고, 줄은 두 묶음 중 하나를 고르기만 합니다
@State private var menuCopy = MenuCopy()
```

즐겨찾기 여부에 따라 두 묶음만 미리 만들어 둡니다. 언어를 바꾸면 그때 한 번 다시 만듭니다. 「언어가 같으면 한 번만 만든다」는 단위 테스트로 확인해 두었습니다.

같은 수정에서 하나를 더 고쳤습니다. 스크롤 중에 자주 바뀌는 값(지금 맨 위에 보이는 절, 화면 위쪽 알약의 글자)을 화면 본문이 직접 읽고 있었습니다. 이러면 그 값이 바뀔 때마다 본문 전체가 다시 그려집니다. 그래서 그 값을 읽는 부분만 작은 하위 뷰로 떼어 냈습니다.

## 결과

같은 기기, 같은 장에서 스크롤이 62~70fps 로 올라왔습니다.

![수정 전과 후의 스크롤 — 같은 장을 같은 조작으로 넘긴 모습](/images/localized-string-per-row-fps/localized-string-per-row-fps-4-before-after.gif)

움짤은 시뮬레이터에서 수정 전 · 후 빌드로 같은 조작을 녹화해 나란히 붙인 것입니다. 위 수치는 이 녹화가 아니라 처음 측정에서 나온 값입니다.


| | 수정 전 | 수정 후 |
|---|---|---|
| 시편 119편 스크롤(iPhone 16e · iOS 26.2) | 평균 약 38fps | 62~70fps |
| 메인 스레드 바쁜 표본 중 문자열 표 읽기 | 약 40% | — |

## 정리

- 현지화 문자열 조회는 공짜가 아닙니다. 특히 bundle 을 직접 넘겨 locale 을 고르는 방식은 부를 때마다 문자열 표를 다시 읽을 수 있습니다.
- 목록 줄처럼 자주 다시 그려지는 곳에서는 문구를 미리 만들어 두는 편이 안전합니다.
- 성능 원인은 짐작으로 정하지 않는 게 좋습니다. 의심 가는 변경은 앞 판과 번갈아 재서 먼저 걸러내고, 그다음 `sample` 같은 도구로 메인 스레드가 실제로 무엇을 하는지 봅니다.

## 참고

- [LocalizedStringResource — Apple Developer Documentation](https://developer.apple.com/documentation/foundation/localizedstringresource)
- [String(localized:) — Apple Developer Documentation](https://developer.apple.com/documentation/swift/string/init(localized:))

이 글의 측정과 원인 추적은 AI(Claude Code)의 도움을 받아 정리했습니다.
