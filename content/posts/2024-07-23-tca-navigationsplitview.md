+++
title = "[TCA] Composable Architecture에서 NavigationSplitView 사용하기"
date = 2024-07-23T16:17:39+09:00
draft = false
tags = ["composablearchitecture", "ios", "navigation", "navigationsplitview", "swift", "swiftui", "tca"]
categories = ["iOS"]
tistory = "https://leetaek.tistory.com/84"
+++

Composable Architecture(이하 TCA)에서 NavigationSplitView를 통해 화면전환을 주절주절 다루는 포스트입니다.

iOS 17.0 이상, TCA 1.9 이상 버전을 기준으로 작성된 코드입니다.

![](/images/tistory/84/01.webp)

![](/images/tistory/84/02.webp)

 *MacOS나 iPadOS에서 많이 사용되는 Sidebar가 있는 네비게이션*

iPad나 MacOS에서 흔히 볼수 있는 네비게이션의 방법 중에, 측면에 네비게이션을 위한 분할 뷰를 열로 생성하는 위와 같은 형태가 있다. 이를 쉽게 구현할 수 있도록 **SwiftUI에서 [NavigationSplitView](https://developer.apple.com/documentation/swiftui/navigationsplitview)라는 API를 지원한다.** NavigationSplitView는 iOS 16.0 이상부터 사용할 수 있다.

그래서 설정 앱을 기준으로 section으로 구분되는 sidebar를 가진 간단한 NavigationSplitView 화면을 구현해보았다.

### Tree-base VS Stack-base

TCA Documents에 따르면**TCA의 네비게이션은 트리 기반과 스택 기반으로 구분**할 수 있다. [각각의 특징과 장단점](https://pointfreeco.github.io/swift-composable-architecture/main/documentation/composablearchitecture/whatisnavigation#Tree-based-vs-stack-based-navigation)은 문서 참고.

나는 다음과 같은 요소들 때문에 둘 중에 **NavigationSplitView는 트리 기반이 더 적합하다고 판단했다.**

1.  Sidebar, Content, Detail의 계층 구조를 가지고 있는 NavigationSplitView는 스택 기반보다는 트리 기반의 네비게이션이 더 유리하게 보였다

2. NavigationSplitView는 Detail에서 **NavigationStackView을 중첩해서 사용도 가능**하다. 때문에 NavigationSplitView는 Sidebar의 항목에 대한 간단한 화면전환을 구현하고, 필요에 따라 경로를 쌓아 복잡한 구조를 구현하는 경우에는 Detail에서 NavigationStack을 중첩해 구현하면 된다.

3. 구현하려고 하는 화면은 한번에 두개의 화면 상태를 가질수 없다. 하나의 Sidebar 항목만을 선택할 수 있도록 상태관리 할 것.

### Reducer

개발자 문서를 참고해 enum으로 상태관리 하는 Tree 기반의 네비게이션을 따랐다.

<script src="https://gist.github.com/LeeTaek/96e3134dffca2395a06ec1ede514952a.js"></script>

Path

한번에 두개의 상태를 가질수 없게 하도록, 네비게이션 경로는 enum으로 설정했다.

<script src="https://gist.github.com/LeeTaek/68f618caa3d83dc513aad0c5c2b802fd.js"></script>

설정 항목 시작시 초기값으로 iCloud 설정 화면으로 지정했다.

Sidebar에서 List의 Selection 파라미터로 Path의 State를 받을건데, 해당 값이 Hashable 프로토콜을 필요로 해서 State를 hashable로 설정했다.

![](/images/tistory/84/03.webp)

*List나 Foreach에서는 아이템 식별과 관리에 Hashable을 요구한다.*

### View

Sidebar 1개의 열을 가지는 NavigationSplitView의 형태이다.

<script src="https://gist.github.com/LeeTaek/30952606bc8f0b2ecdbb514d4fb086af.js"></script>

Detail

Sidebar View는 고정되지만, Detail은 Reducer에 따라 변화될 수 있기 @ViewBuilder메소드로 분리했다.

switch를 통해 store의 path에 따라 view가 전환되도록 설정했다.

<script src="https://gist.github.com/LeeTaek/8017de87fc8ef0cb606d06ff6281b794.js"></script>

사실은 .navigationDestination modifier를 통해 설정하려고 했는데. 요렇게 한 이유는 애먹은점에서 추가 설명.

Sidebar

<script src="https://gist.github.com/LeeTaek/21591976d7fa6bcbf9b8c8285082a354.js"></script>

NavigationLink를 통해, 선택한 값을 push로 보내서 처리할 수 있도록 했다.

### 애먹은 점

Sidebar에서 주석에 번호를 단 것으로 코드설명을 조금 더 하면

1. List 파라미터
List의 selection 파라미터를 통해 선택된 List 항목을 표시할 수 있다. selection에 넘겨줄 @State 변수를 따로 하나를 만들어서 선택된 값을 관리할 수도 있지만**store에 이미 선택된 값을 가지고 있는데, 이중으로 만들어 상태관리하고 싶지 않았다.**

그러나 TCA는 단방향 아키텍처이기 때문에 action을 통해 reducer에서만 상태값을 변경할 수 있다.

![](/images/tistory/84/04.webp)

*직접적으로 state에 binding을 하려고 하면 get-only라는 에러가 든다.*

View에서 바인딩이 필요한 경우, sending이라는 메서드를 통해 미리 store에 미리 지정된 action으로 변경된 값을 처리할 수 있다. 바인딩을 위한 action도 reducer에 추가해주었다.

<script src="https://gist.github.com/LeeTaek/49569f9d3e94f70b522ed5a487a7c8be.js"></script>

2. List 내부 구현
사실 가장 애먹었던 부분인데, 최종적으로는 NavigationLink를 사용한 방법으로 이용했다.

원래는 Reducer에 각 뷰로 이동하는 action case를 설정하고, Button으로 해당 액션들을 store로 보내 실행하려고 했었다.

코드로 보면 다음과 같다.

<script src="https://gist.github.com/LeeTaek/365cfdb6124e38a73196853c9c018b16.js"></script>

<script src="https://gist.github.com/LeeTaek/ea7859979d989e5cdced03686784b700.js"></script>

그런데 이 경우 state.path가 설정된 후에 바로 nil로 재설정 되는 문제가 생겼다.  원인을 보자면.

- Button눌러 실행되는 action을 통해 reducer에서 state.path를 할당된다.
- 그런데 Path.State의 해시값과 Button에 tag로 설정한 Path.State의 해시값이 같지 않았다.
- List는 이에 선택된 Tag가 없다고 판단하고 selection에 nil을 보내게 되고, 이를 reducer의 push action에서 받아 다시 nil로 처리하는 문제였다.

**List에서 선택한 값을 push 액션을 보내 설정하면 되기 때문에 코드에서 사용한 NavigationLink(_ titleKey, value: )을 사용하던, 혹은 Button 대신 Text으로 교체해 .tag를 붙여 사용하던 잘 동작한다.**

3. navigationDestination Modifier

NavigationSplitView 관련해서 알아보던 중에 [navigationDestination modifier](https://developer.apple.com/documentation/localauthentication/localauthenticationview/navigationdestination(item:destination:)/#discussion)를 이용하는 방법이 있었다.

iOS17.0 이상부터는 해당 api를 NavigationSplitView에서도 사용할 수 있다는 것이었다.

<script src="https://gist.github.com/LeeTaek/9aa33e0a3c558ab7fd1b8641d340b705.js"></script>

위에서 설명한대로 NavigationLink와 Text 둘 다 동일하게 잘 동작한다.

그런데 Detail에서 세부적인 네비게이션을 구현하기 위해 NavigationStack을 중첩하여 사용하는 경우에 문제가 생겼다.

<script src="https://gist.github.com/LeeTaek/bc53005aaa5d7d359f00c814df60e3a0.js"></script>

![](/images/tistory/84/05.webp)

예제 코드에서는 중첩된 코드를 Stack-base로 만들었다.

런타임 에러의 설명을 보면 **SwiftUI에서는 NavigationLink를 통해 NavigationStack이 포함된 뷰를 직접적으로 여는것을 허용하지 않는다**는 내용이었다.  이 에러는 NavigationLink가 아닌 Text로 구현해도 동일하게 발생했다.

아마 NavigationSplitView의 Sidebar에서 내부적으로 NavigationLink를 활용하기 때문에 그 안에 NavigationStack을 포함한 뷰를 .navigationDestination을 통해 전달해서 이런 메세지가 뜨는게 아닌가 싶다.

때문에 이를 회피하고자 Detail을 분리해 iflet으로 view를 처리했다.

![](/images/tistory/84/06.gif)

*완성~~~*

###

### 최종 코드

<script src="https://gist.github.com/LeeTaek/948dfa911c36fa461e5deb87072c6151.js"></script>

### 참조

[https://developer.apple.com/documentation/swiftui/navigationsplitview](https://developer.apple.com/documentation/swiftui/navigationsplitview)

[https://pointfreeco.github.io/swift-composable-architecture/main/documentation/composablearchitecture/navigation](https://pointfreeco.github.io/swift-composable-architecture/main/documentation/composablearchitecture/navigation)
