+++
title = "[TCA] SharedState의 UserDefaults 확장하기 (CodableAppStorageKey)"
date = 2024-07-18T10:20:18+09:00
draft = false
tags = ["composablearchitecture", "github", "ios", "swift", "swiftui", "tca", "xcodecloud"]
categories = ["iOS"]
tistory = "https://leetaek.tistory.com/82"
+++

TCA에서 사용되는 [SharedState](http://Sharing%20state Learn techniques for sharing state throughout many parts of your application, and how to persist data to user defaults, the file system, and other external mediums.  Overview Sharing state is the process of letting many features have access to the same data so that when any feature makes a change to this data it is instantly visible to every other feature. Such sharing can be really handy, but also does not play nicely with value types, which are copied rather than shared. Because the Composable Architecture highly prefers modeling domains with value types rather than reference types, sharing state can be tricky.  This is why the library comes with a few tools for sharing state with many parts of your application. There are two main kinds of shared state in the library: explicitly passed state and persisted state. And there are 3 persistence strategies shipped with the library: in-memory, user defaults, and file storage. You can also implement your own persistence strategy if you want to use something other than user defaults or the file system, such as SQLite.)에 대한 내용입니다.

여기서는 SharedState이 무엇인지 소개하는 내용 보다도 사용하면서 조금은 편하게 쓰고자 애쓴 흔적들을 **주절주절** 적습니다.

### SharedState

어느날 갑자기 TCA 제작사인 Point-Free 에서 메일이 겁나 날라왔다.

TCA 새로운 에피소드, 바로 Sharing State입니다~~~ 하는 내용이었다.

![](/images/tistory/82/01.webp)

*꾸준히 날라오는 그들의 에피소드 ...*

예전에 Github에서 SharedState에 대한 논의가 활발히 다뤄지는 것만 봤었다. 느낌상 큰 업데이트일것 같아서 [Shared-State-beta 브랜치](https://github.com/pointfreeco/swift-composable-architecture/tree/shared-state-beta)에서 개발되고 있는 코드를 사이드 프로젝트에 미리 적용봤었었다.

SharedState는 ChildFeature로 상태를 굳이 하나하나 전달하지 않아도 되는, 왕편리함 그 자체였다.

그 중에 자주 사용하게 되는 부분이 TCA에서 제공해주는 Persisted SharedStated의 [UserDefaults](https://pointfreeco.github.io/swift-composable-architecture/main/documentation/composablearchitecture/sharingstate/#User-defaults)였다.

정말 편한데 UserDefaults의 단점은 **간단한 자료형**들만 사용 가능하다는 것이었다.

![](/images/tistory/82/02.webp)

*간단한게 아니면 다른걸 써라 ...*

그래서 UserDefaults에 좀 더 복잡한 자료형(사실 그렇게 복잡한것도 아닌데...)들도 저장하기 위해 찾다가 감사하게도[Github에 누군가 올려둔 것을 발견했다.](https://github.com/pointfreeco/swift-composable-architecture/discussions/2857#discussioncomment-9087925)

코드의 내용은 UserDefaults에 기존 자료형 저장하던 것 처럼, **Data형태로 encoding/deconding 저장하는 것**이었다.

### CodableAppStorageKey

<script src="https://gist.github.com/LeeTaek/3c4b20d4d34b1f6e3c45952cd1fe711e.js"></script>

간단하게 코드를 분석해보면,

1. Generic으로 Codable을 준수하는 자료형이 들어오면 이를 CodableAppStorageKey를 통해 처리하도록 [PersistenceReaderKey에 등록한다.](https://github.com/pointfreeco/swift-composable-architecture/blob/shared-state-beta/Sources/ComposableArchitecture/Documentation.docc/Articles/SharingState.md#custom-persistence)

2. CodableAppStorageKey는 들어온 값을 Data 형태로 encoding/decoding하여 이미 **TCA에 구현되어 있는 AppStorageKey를 통해 넘겨, UserDefaults로 save, load, subscribe 한다.**

사용하려면 물론 Codable 프로토콜을 준수하는 객체를 보내야 한다.

위 코드는 매우 효과적으로 잘 동작했었다. 분명히 잘 동작을 했었다...

### 문제 발생

어느날 작업한 내용을 Push하고 업무를 보고 있었는데 Xcode Cloud에서 메일이 하나 날라왔다.

![](/images/tistory/82/03.webp)

*?!???*

빌드 자체가 안된댄다 ...

분명 실기기 테스트까지 하고 올렸는데 빌드가 안된다니, cloud로 올린 코드를 확인해봤는데 잘만 돌아갔다.

그래서 로그를 따라서 Composable architecture 라이브러리의 AppStorageKey를 따라가봤는데 생성자도 로그랑은 다르게 public으로 선언되어있었다.

환경상의 문제가 있나 싶어 Xcode Cloud의 환경도 확인해봤는데 별다른 문제가 없어보였다.

그러다가 생각났다.

![](/images/tistory/82/04.webp)

![](/images/tistory/82/05.webp)

 *beta 브랜치와 main브랜치에 적용된 생성자...*

내가 앱에 적용했던것은 shared-state-beta 브랜치였다는 것을...

**SharedState가  main으로 merge되면서 생성자의 접근제어자가 filePrivate로 변경되어 있었다.**

AppStorageKey를 Init해 사용하는 CodableAppStorage은 애초에 빌드가 될수 없었던 것이었다.

결국 기존에 구현된 AppStoragekey를 참고해서 다시 만들었다.

아래는 기존에 구현된 코드를 바탕으로 무식하게 따라 만든 CodableAppStorageKey이다 ...

### 다시 만든 MusicCodableAppStorageKey

위의 Music은 음악이 아니라 무식입니다...

<script src="https://gist.github.com/LeeTaek/b9a3f73d7db5282596a27212b7d28994.js"></script>

큰 틀은 동일하다. Codable한 객체를 받아서 Data 타입으로 encoding/decoding 한 후, 이를 UserDefaults로 저장하는 것이다.

주석으로 쓴 번호를 따라 간단하게 코드 설명을 하면

1. AppStorage로 Codable한 값을 받을 경우 CodableAppStorageKey로 처리하도록**PersistenceReaderKey를 등록**한다.

2. **PersistenceKey protocol을 준수**하는 struct를 만든다. 이전에는 이미 정의된 AppStorageKey를 그대로 가져와 사용했기 때문에 프로퍼티로 AppStorage를 가져와 encoding/decoding한 Data를 넘겨줘 처리했지만, 이제는 AppStorageKey를 못가져오기 때문에 AppStorageKey의 init을 못사용하기 때문에 저장에 사용할 key값과 store를 그대로 정의한다. store는 init에서 swift-dependencies로 가져와 사용한다.

3-5. AppStorageKey의 경우 save, load를 구현하는 Lookup이라는 프로토콜을 사용했다. 들어오는 데이터 타입에 따라 적합한 Lookup을 통해 로직을 구현했다. 나는 Data를 저장하는 부분만 필요해 해당 부분이 구현된 **CastableLookup을 가져와서 encoding/decoding 과정만 추가**했다. Subscribe는 그대로 활용.

사용은 SharedState의 userDefaults와 동일하게 사용하면 된다.

![](/images/tistory/82/06.webp)

*Codable을 준수하는 PencilPalatte를 appStorage로 저장하고 있다.*

![](/images/tistory/82/07.webp)

*xcode cloud에서도 잘 처리가 됐다.*

물론 복잡한 자료나 대규모 데이터를 다루는 경우라면 fileStorage를 쓰는게 이득이겠지만, 그게 아니고 Codable만 준수한다면 편하게 AppStorage를 사용해 저장할 수 있다.

### 참고

[TCA ShardState Docs](https://pointfreeco.github.io/swift-composable-architecture/main/documentation/composablearchitecture/sharingstate/#Overview)

[Documentation   pointfreeco.github.io](https://pointfreeco.github.io/swift-composable-architecture/main/documentation/composablearchitecture/sharingstate/#Overview)

[TCA SharedState-beta Discussions](https://github.com/pointfreeco/swift-composable-architecture/discussions/2857#discussioncomment-9087925)

[Shared state beta · pointfreeco swift-composable-architecture · Discussion #2857 Hello all, today we are beginning a very exciting beta for the Composable Architecture. It brings all new tools to the library for sharing state in your application and persisting data. To try out ... github.com](https://github.com/pointfreeco/swift-composable-architecture/discussions/2857#discussioncomment-9087925)
