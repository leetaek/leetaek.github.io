+++
title = "[AI] Codex - Worktree와 Local"
date = 2026-04-02T18:01:00+09:00
draft = false
tags = ["ai", "automation", "chatgpt", "codex", "github", "ios", "자동화", "프롬프트"]
categories = ["AI"]
tistory = "https://leetaek.tistory.com/90"
+++

잘못된 내용이거나 수정사항이 있다면 알려주세욥 ...!

Codex를 쓰다 보면 같은 작업이어도 **Local에서는 잘 되는데 Worktree에서는 막히는 경우**가 있다.

![worktree에서 테스트 돌렸다 발생한 에러](/images/tistory/90/01.webp)

*worktree에서 테스트 돌렸다 발생한 에러*

**먼저 결론**

- **코드 수정, 브랜치 분리, PR 생성 중심 작업** → **Worktree**가 적합하다.
- **Xcode, Simulator, 로컬 개발 환경에 의존하는 검증 작업** → **Local**이 적합하다.

Worktree는 **격리된 작업 공간**, Local은 **내가 실제로 개발하는 환경**이라고 이해했다.

---

## 왜 이런 차이가 생기나 ?

**Worktree**

Codex 공식 Troubleshooting 문서의 [**Code doesn’t run on a worktree**](https://developers.openai.com/codex/app/troubleshooting#code-doesnt-run-on-a-worktree)섹션에서는,

worktree가 **기존 로컬 프로젝트와는 다른 디렉토리**에 생성되며, **Git에 체크인된 파일들만 상속**한다고 설명한다.

저장소에 포함되지 않은 로컬 설정이나 의존성, 개발 중 생성된 파일들은 그대로 따라오지 않을 수 있다.

그래서 프로젝트에 따라서는 worktree에서 **별도의 setup script를 다시 실행해야 할 수 있고**,

대안으로는 기존의 regular local project에서 변경사항을 checkout해서 작업할 수도 있다고 안내한다.

**Local**

Codex 공식 [**Local environments**](https://developers.openai.com/codex/app/local-environments) 문서에서도 비슷한 맥락을 설명한다.

worktree는 local task와 **다른 디렉토리에서 실행되기 때문에**, 프로젝트가 완전히 셋업되지 않았을 수 있고,

저장소에 체크인되지 않은 의존성이나 파일이 빠질 수 있다고 한다.

그래서 Codex는 worktree를 만들 때 필요한 환경을 맞추기 위해 **setup scripts를 함께 실행하는 방식**을 지원한다.

공식 문서 기준으로 정리하면 이렇게 이해할 수 있다.

- **Worktree** → 브랜치 분리, 코드 수정, PR 생성 같은 작업에 유리, Git 기준으로 격리된 병렬 작업 공간
- **Local** → Xcode, Simulator, 런타임, 체크인되지 않은 파일, 빌드 아티팩트에 민감한 작업에 더 유리할 수 있음

---

## Worktree가 잘 맞는 경우

Worktree는 현재 작업 중인 로컬 변경과 분리된 별도 공간에서 Codex가 작업하도록 해준다.

그래서 아래 같은 작업에 특히 잘 맞는다.

- 작은 기능 구현
- 리팩토링
- 테스트 코드 작성
- 문서 수정
- 브랜치 생성, 커밋, Draft PR 생성
- 현재 작업 중인 변경사항과 섞이면 안 되는 자동화

예를 들어 이전에 포스팅한 nightly work처럼 “이슈 하나 처리하고 브랜치 따서 PR까지 만들기” 같은 Git 중심의 흐름은 Worktree가 편하다.

**Worktree의 장점**

- 현재 작업 중인 변경사항과 충돌이 적다
- 자동화에 올리기 좋다
- 브랜치/PR 단위 작업이 깔끔하다
- 실험적인 변경을 격리해서 다룰 수 있다

**Worktree의 한계**

하지만 Worktree는 **내 로컬 개발 환경 그 자체**는 아니다.

그래서 아래 같은 작업에서는 한계가 드러날 수 있다.

- Xcode가 실제로 열려 있어야 하는 경우
- iOS Simulator 접근이 필요한 경우
- CoreSimulatorService에 의존하는 테스트
- 로컬에 이미 구성된 개발 환경을 그대로 써야 하는 경우
- 앱 실행, 디버깅, UI 확인이 필요한 경우

---

## Local이 잘 맞는 경우

Local은 내가 지금 실제로 쓰고 있는 프로젝트 디렉터리와 환경에서 Codex가 작업하는 방식이다.

즉, 내가 직접 개발하던 맥 환경을 그대로 활용할 수 있다.

그래서 아래 같은 작업은 Local이 훨씬 안정적이다.

- xcodebuild test
- iPad/iPhone Simulator 기반 테스트
- Xcode, DerivedData, Simulator runtime 등 로컬 환경을 활용하는 작업
- 직접 앱을 띄워 확인해야 하는 작업
- 기존 workspace, scheme, simulator 상태를 그대로 써야 하는 작업

특히 iOS 프로젝트에서 테스트 코드는 Worktree에서도 잘 만들지만, 실제 test run은 시뮬레이터 검증은 Local이 더 안정적으로 돌아가는 것 같다.

**Local의 장점**

- 현재 개발 환경과 가장 가깝다
- Xcode/Simulator 의존 작업에 강하다
- 실제 빌드/테스트 재현성이 높다
- 로컬에서 이미 되던 흐름을 그대로 가져갈 수 있다

**Local의 한계**

- 현재 작업 중인 변경사항을 직접 건드릴 수 있다
- 자동화 시 격리가 덜 된다
- 실험적 변경과 기존 작업이 섞일 수 있다

---

## 결론

- Xcode가 실제로 열려 있어야 하거나
- Simulator 상태를 그대로 활용해야 하거나
- DerivedData, runtime, local dependency 상태에 영향을 받는 작업

은 Worktree보다 **Local 환경이 더 안정적으로 동작할 가능성이 높다.**

반대로 코드 수정 자체나 브랜치/PR 중심의 자동화는 Worktree가 더 적합하다.

### reference

<https://developers.openai.com/codex/learn/best-practices?utm_source=chatgpt.com>

[Best practices – Codex | OpenAI Developers Getting started with Codex and proven practices for better results developers.openai.com](https://developers.openai.com/codex/learn/best-practices?utm_source=chatgpt.com)

- [Best practices – Codex | OpenAI Developers](https://developers.openai.com/codex/learn/best-practices?utm_source=chatgpt.com) — developers.openai.com
