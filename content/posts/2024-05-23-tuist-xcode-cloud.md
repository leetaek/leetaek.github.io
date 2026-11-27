+++
title = "[Tuist] Xcode Cloud 적용하기 (SwiftLint, FirebaseCrashlytics)"
date = 2024-05-23T17:50:53+09:00
draft = false
tags = ["tuist", "xcodecloud", "ios", "ci-cd", "swiftlint", "firebase"]
categories = ["iOS"]
tistory = "https://leetaek.tistory.com/81"
+++

사이드 프로젝트에 Xcode Cloud를 적용해보면서 겪은 과정을 정리한 글입니다.
Xcode Cloud에 대한 소개와 사용법 보다는 Tuist로 관리되는 프로젝트를 Xcode Cloud 환경에 맞게 설정한 내용을 기록하기 위해 적었습니다.

#### Xcode Cloud

Tuist로 관리하는 프로젝트를 실행할 때에 tuist install, tuist generate와 같은 명령어가 필요하다. Tuist를 통해 관리하는 프로젝트는 Xcode Cloud를 어떻게 적용시킬까? 다행히 Xcode Cloud에서는 빌드 스크립트를 작성해서 필요한 설정을 구성할 수 있다.

![Xcode Cloud에서 지원하는 build script 실행 시점](/images/tistory/81/01.webp)

*Xcode Cloud에서 지원하는 build script 실행 시점*

post-clone 타이밍에 Tuist의 설치와 실행 명령어를 스크립트로 작성하면 될 것 같다. Xcode Cloud의 Custom Build Script를 사용하는 방법은 **프로젝트의 루트경로에 ci_scripts라는 폴더를 만들고, ci_post_clone.sh 파일을 생성해 원하는 스크립트를 작성하면 된다.** 그럼 Xcode Cloud가 워크플로우 수행할 때에 해당 스크립트를 실행하게 된다.

#### ci_post_clone.sh의 내용

Tuist 의 버전이 4.0 이상으로 올라가면서 이제는 **mise로 설치할 것을 권장한다.** mise로 설치하게 되면서 Tuist 의 버전 트래킹이 용이하게 되는 장점이 생긴다. mise로 Tuist를 설치했다면 프로젝트 루트 경로에 자동으로 .mise.toml파일이 생성되어 있을 것이다.  .mise.toml 파일에 mise를 통해 설치한 도구의 버전이 기록되기 때문에 **.tuist-version 파일을 더이상 사용하지 않아도 된다.**

![.mise.toml 파일](/images/tistory/81/02.webp)

*.mise.toml 파일*

때문에 Xcode Cloud의 환경에도 mise를 설치하고, .mise.toml을 참조해 설치된 Tuist를 실행하면 버전을 굳이 명시하지 않아도 tuist가 정상 동작한다. 각 코드는 대략적으로 주석을 달아놨으니 참고하면 될 것 같다.

<script src="https://gist.github.com/leetaek/b1a5a58ddeef77ceacc61ea8d3c5dbeb.js"></script>

#### SwiftLint와 FirebaseClashlytics

그런데 내 경우 Tuist 설정까지는 잘 동작했는데 Xcode Cloud 환경에서 Swiftlint와 FirebaseClashlytics에서 "command phasescriptexecution failed with a nonzero exit code" 같은 에러가 떴다. 두 라이브러리를 적용하며 Build Phase에 작성한 스크립트에서 발생한 에러였다. 두가지 원인이었는데

1. Swiftlint는 기존에 Homebrew로 설치해서 사용했었다.  -> mise 말고 homebrew도 설치해야하나?

2. FirebaseCrashlytics의 스크립트 경로 설정이 잘못되었다.  -> 경로를 재설정해주면 그만

그런데 고맙게도 **mise에서 SwiftLint도 지원해준다!** mise를 통해 swiftlint를 설치하면 .mise.toml에 자동으로 기록된다. 그러면 Xcode Cloud 환경에도 SwiftLint가 설치되기 때문에 homebrew가 아닌 mise를 사용하도록 변경했다. 터미널에서 다음 명령어를 통해 Mise로 SwiftLint를 설치할 수 있다.

<script src="https://gist.github.com/leetaek/99cb963d2147cb4f45ce56c9def4cccc.js"></script>

그리고 brew로 사용하던 SwiftLint를 Mise를 통해 사용하도록 Build Phase의 스크립트도 변경해주었다. 아래는 프로젝트의 구성 경로와 해당 경로를 기준으로 Tuist edit에서 작성한 스크립트 내용이다. SwiftLint의 룰 파일인 .swiftlint.yml는 프로젝트의 루트 경로에 있다.

![](/images/tistory/81/03.webp)

![](/images/tistory/81/04.webp)

*프로젝트 파일 경로*

<script src="https://gist.github.com/leetaek/dec6b2e498f21d4b7699ec4fc4c93747.js"></script>

#### 결과

![Xcode Cloud 환경에서 성공적으로 진행됐다.](/images/tistory/81/05.webp)

*Xcode Cloud 환경에서 성공적으로 진행됐다.*

![그리고 도착한 Success 메일](/images/tistory/81/06.webp)

*그리고 도착한 Success 메일*

성공적으로 빌드되었다. 스크립트 작성 때문에 16번정도 삽질한 것 같다. Success 메일이 날라왔을 때 왕감격스러웠다 ㅠ_ㅠ

#### 결론

- 기존의 다른 분들이 작성했던 글들을 보면 Tuist가 Mise를 환경설정에 훨씬 용이해진 것 같다.

- CI/CD를 처음 적용해봤는데 테스트 플라이트, 배포 등이 경험하게 될 장점이 기대된다. (1인으로 진행하는 프로젝트지만...)

- 터미널 경로를 잘 확인하자. ci_scripts경로에서 tuist generate가 실행되지 않아서 무지하게 당황해하며 진땀뺐다....

#### Reference

<https://docs.tuist.io/guide/introduction/installation>

- [Installation | Tuist](https://docs.tuist.io/guide/introduction/installation) — docs.tuist.io

<https://developer.apple.com/documentation/xcode/writing-custom-build-scripts>

- [Writing custom build scripts | Apple Developer Documentation](https://developer.apple.com/documentation/xcode/writing-custom-build-scripts) — developer.apple.com

<https://github.com/klundberg/asdf-swiftlint>

- [GitHub - klundberg/asdf-swiftlint: An asdf plugin for swiftlint](https://github.com/klundberg/asdf-swiftlint) — github.com

<https://github.com/tuist/tuist/issues/5863>

- [Xcode Cloud + Mise doesn't work · Issue #5863 · tuist/tuist](https://github.com/tuist/tuist/issues/5863) — github.com

<https://green1229.tistory.com/351>

- [Tuist를 통해 SwiftLint 사용하기](https://green1229.tistory.com/351) — green1229.tistory.com

<https://nsios.tistory.com/190>

- [[Swift] Xcode Cloud(CI/CD) + Tuist(프로젝트관리툴) + dSYMs 업로드까지 자동화 배포하기 (feat. 스크립트쉘)](https://nsios.tistory.com/190) — nsios.tistory.com
