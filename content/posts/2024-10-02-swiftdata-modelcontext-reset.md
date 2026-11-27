+++
title = "[SwiftData] iOS18이상에서 ModelContext.reset 호출로 인해 나타나는 런타임 오류 (TCA+SwiftData)"
date = 2024-10-02T14:45:12+09:00
draft = false
tags = ["ios", "ios18", "swiftdata", "swiftui", "tca", "uikit", "개발", "모바일"]
categories = ["iOS"]
tistory = "https://leetaek.tistory.com/85"
+++

![](/images/tistory/85/01.webp)

> Fatal error: This model instance was destroyed by calling ModelContext.reset and is no longer usable.

####

####

#### 발생 상황

iPadOS18 이상으로 업데이트하니 갑자기 사이드 프로젝트 앱에서 다음과 같은 에러와 함께 강제 종료가 되기 시작했다.

iOS17까지는 잘 돌아가던 코드가 18부터 오류가 발생하기 시작한 것.

앱은 TCA 아키텍처 구조에서 SwiftData를 사용하고 있었다.

사용하고 있던 방법은 다음과 같다.

1.  SwiftData의 데이터를 관리하고 저장소에 접근하기 위한 ModelContainer를 생성

<script src="https://gist.github.com/LeeTaek/5bf0e59369711f5b2079bc03f9e15503.js"></script>

2. 동시성 문제 없이 데이터를 관리하기 위해 ModelActor로 데이터를 관리.

<script src="https://gist.github.com/LeeTaek/6b0ea0785bc637d2a9d0e9adb98f0be3.js"></script>

  2-1. 후에 재사용할 수 있을거 같아서 해당 Actor 등은 위해 모듈로 분리함.

  2-2. 재사용성을 위해 Actor의 CRUD 메서든는 제네릭으로 모델 타입을 받도록 구현

3. 구체적인 모델의 CRUD를 위한 struct를 구현. struct는 TCA의 Dependencies로 등록후 사용.

<script src="https://gist.github.com/LeeTaek/644a6f35b00e811b40e8664406a8df0d.js"></script>

3번의 구현에서 불편한 부분이 있으시다면 그 부분이 원인 맞습니다...

#### 문제의 원인

SwiftData에 대한 자료가 많지 않아서 찾을수 있을까 싶었지 이미 이 내용에 대한 포럼이 열려있었다.

결론적으로 보면 ModelActor를 사용할 때에 **모델 인스턴스가 액터보다 더 오래 살아있는 경우**발생할 수 있는 에러였다.

**오히려 iOS17에서 정상동작 하는것이 버그**였다고 한다.

왜 모델의 인스턴스가 액터보다 오래 살아있는 경우 문제가 생길까?

1. actor의 역할

 actor는 모델의 인스턴스에 대한 접근을 큐를 통해 직렬화 해 순차적으로 진행한다. 때문에 동시접근이나 불일치를 방지하여 데이터에 안전한 접근을 보장한다. ModelActor는 모델의 인스턴스를 관리하는 주체로 여러 스레드에서 동일한 인스턴스에 접근할 때에 발생할 수 있는 문제를 예방한다.

2. 모델 인스턴스의 생명주기

모델의 인스턴스는 액터에 연결된 ModelContext에서 가져온 데이터이다. 모델의 인스턴스는 ModelContext가 살아있을 때에 유효하다. **actor가 존재하지 않는다면 ModelContext도 해제되고, 모델 인스턴스도 함께 해제된다.**

![](/images/tistory/85/02.webp)

*modelContext를 통해 인스턴스를 가져온다.*

3. 생명주기로 인해 발생할 수 있는 문제

- **런타임 오류**: 인스턴스가 actor보다 오래 살아남으면, 인스턴스에 접근할 때에 해제된 객체에 접근하여 런타임 오류가 발생할 수 있다.
- **데이터 무결성**: 혹여나 actor가 해제된 상태에서 값에 접근하여 수정이 된다면, actor가 관리하던 ModelContext의 상태와 불일치해서 데이터의 무결성이 보장되지 않을 수 있을 것 같다.

- **메모리 누수**: 사용하지 않는 actor에 대한 참조가 남아 메모리 누수로 이어질 수도 있을 것 같다.

#### 해결 방안

이를 해결하기 위해서 ModelContainer나 actor를 singleton으로 구현하는 것을 추천하고 있다.

그런데 나는 ModelContainer는 이미 singleton이고, actor의 경우도 한 상황에 Drawing 모델에 대한 인스턴스에만 접근하는데 왜 에러가 떴을까...?

![](/images/tistory/85/03.webp)

*메서드들에 actor가 지역변수로 선언되어 있다.*

정답은 구체화한 struct의 메서드에 있었다.

1.  각 메서드 내에 modelActor가 지역 변수로 선언된다. 이 actor는 메서드의 생명주기와 함께하며, 메서드 종료시 ModelContext도 함께 해제된다.

2. 메서드가 종료된 후에 모델 인스턴스에 접근하려고 하면, 유효하지 않은 메모리 접근으로 런타임 에러가 발생된다.

<script src="https://gist.github.com/LeeTaek/031bf850fae3e1b03b83b205300b6201.js"></script>

때문에 해당 액터를 지역 변수가 아니라 전역 변수로 선언하면 actor의 생명주기는 struct가 존재하는 한 유지되며 해당 문제는 해결된다.

사실 메서드들에서 중복으로 사용하는 변수들이기 때문에 진작에 전역변수로 빼놨으면 이런 문제가 발생하지도 않았을 일이긴 하다...

####

#### 결론

- 돌아가던 코드가 안돌아간다고 버그가 아니다. 오히려 돌아갔던게 버그일 수 있다.

- **객체의 생명주기를 고려하며 로직을 설계하자.**

- 공통코드를 왜 반복해서 작성함?...?

- 문제의 원인은 생각보다 간단할 수 있다. 자료가 적은 최신 API라 하더라도 당황하지 말자.

#### 참조

[https://forums.developer.apple.com/forums/thread/757521](https://forums.developer.apple.com/forums/thread/757521)

[iOS 18 SwiftData ModelContext reset | Apple Developer Forums I have the exact same problem. Sometimes when I build and run the app, it crashes and shows me the error. This happens randomly . Sometimes it builds fine and sometimes it crashes. Also, when it crashes and I relaunch the app without building again, it nev forums.developer.apple.com](https://forums.developer.apple.com/forums/thread/757521)

[https://forums.developer.apple.com/forums/thread/764281](https://forums.developer.apple.com/forums/thread/764281)

[iOS 18 strange SwiftData fatal err… | Apple Developer Forums Yeah, the code you provided in your comment creates the model container and holds it with a local variable, which is released after fetchEvents is done, which triggers the error when you try to access the events. Your new code seems to hold the model conta forums.developer.apple.com](https://forums.developer.apple.com/forums/thread/764281)
