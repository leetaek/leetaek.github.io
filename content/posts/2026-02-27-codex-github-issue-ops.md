+++
title = "[AI] Codex 자동화를 위한 GitHub Issue 운영 체계 만들기 (라벨/템플릿/분류)"
date = 2026-02-27T16:34:52+09:00
draft = false
tags = ["ai", "automation", "chatgpt", "codex", "github", "ios", "에이전트코딩", "자동화"]
categories = ["AI"]
tistory = "https://leetaek.tistory.com/87"
+++

이번 글은 “Codex Automation”을 활용한 에이전트 코딩 관련 포스트이면서도, 사실은 그보다 앞단의 이야기다.

글은 다음과 같이 세 편으로 나눠 정리하려고 한다.

1. **[AI]  자동화를 위한 GitHub Issue 운영 체계 만들기 (라벨/템플릿/분류)**
2. [[AI] GitHub Actions + BigQuery로 Nightly Report 자동 수집 파이프라인 구축하기](/posts/2026-03-06-nightly-report-bigquery/) (작성중 ...)
3. [AI] Codex Automation으로 “이슈 1개 → Draft PR” 밤새 돌려보기 (worktree/sandbox/gh/prompt)

요번 글은 첫번째 글이다.

---

## 왜 자동화가 필요했나?

회사 일이 바빠지면서, 1인으로 진행하던 사이드 프로젝트가 생각보다 빠르게 소홀해졌다.

코드를 아예 안 보는 건 아닌데, ‘기능 추가’는커녕 인지하고 있는 버그와 레포트로 들어오는 버그가 산더미처럼 쌓이기 시작했다.

특히 내가 힘이 빠졌던 지점은 ‘개발이 어렵다’기보다,

퇴근 후 남는 에너지로는 사이드 프로젝트를 제대로 굴리기가 어렵다는 점이었다.

집에 돌아와 저녁 먹으면 9시, 집안일 마치면 11시.

그 시간에 사이드 프로젝트를 켜면 “오늘은 어디부터 해야 하지?”에서 이미 지친다.

테스트 공백에 대한 부담도 그중 하나였다.

버그를 하나 고치고도 ‘이거 다른 데 망가뜨린 건 아닐까?’ 하는 찝찝함이 남고,

그 찝찝함이 결국 다음 작업의 진입장벽이 됐다.

그래서 바랐던 건 “밤새 다 고쳐주기”가 아니라,

**발견된 이슈나 버그를 정리해두고, 가능하면 초안(PR) 까지 만들어두는 흐름**이었다.

자동화를 통해 밤 시간에는 반복 작업(이슈 수집/정리, PR 초안)을 맡기고,

나는 낮에 결과를 한 번에 모아 **리뷰하고 결정하고(필요하면 테스트까지)** 처리하는 식으로 에너지를 분리하고 싶었다.

![Codex Automation](/images/tistory/87/01.png)

*Codex Automation*

그때 발견한 게 Codex Automation이었다.

CLI로 사용하던 **Codex가 최근 MacOS용 앱을 출시했는데 함께 소개된 기능이  Automation 이다.**

이 기능을 적극적으로 사용해보고 싶었다.

## 기대하는 구조

![Codex Automation이 GitHub 에 올린 Issue를 읽고 수정해서 올린 PR](/images/tistory/87/02.png)

*Codex Automation이 GitHub 에 올린 Issue를 읽고 수정해서 올린 PR*

내가 기대하는 구조는 간단하다.

밤 시간에 자동화가 할 수 있는 일을 맡기고, 나는 낮에 결과물을 한 번에 모아 리뷰하고 결정만 하는 방식.

일단 “개발을 대신한다”기보다는 영향이 크지 않은 버그 위주로 정리해주는 작업을 기대했다.

플로우는 다음과 같다.

> 1. 앱에서 개선할 점들을 GitHub Issue에 수집
>    (1-2) 개발자가 직접 개선 방향/할 일을 이슈로 수동 작성
>    (1-1) analytics 기반 오류 로그를 Nightly report로 자동 수집
> 2. Codex Automation이 우선 순위에 따라 이슈 하나를 골라 밤새 작업
> 3. 작업한 내용을 PR로 남김
> 4.개발자가 시간 될 때 PR 확인

이 방식이 되려면, 무엇보다 이슈가 잘 모이고 잘 정리되어 있어야 했다.

그래서 버그/개선 포인트를 자동으로 모아 GitHub Issue로 쌓는 파이프라인부터 만들기로 했다.

이번 포스트에서 다루는 아래 내용은 Github Issue 수집에 해당하는 1번 내용이다.

---

## GitHub Issue label 정리

파이프라인을 구성하기 전에 Agent가 수행할 작업을 결정하는데 판단 기준이 되도록 label을 정리하는게 적절하다 생각이 들었다.

그래서 아래와 같이 라벨을 정리했다.

![정리한 Label](/images/tistory/87/03.png)

*정리한 Label*

1. **우선순위: 작업을 결정하는 우선순위**
   - prio:p0 매우 시급함(사용성/핵심 기능 영향)
   - prio:p1 빠른 시일 내
   - prio:p2 여유 있을 때
2. **상태**
   - status:needs-repre: 아직 증거/재현 정보 부족(추가 로그 필요)
   - status:ready 바로 작업해도 되는 상태
   - status:in-progress: 작업 진행 중
3. **작업 범위**
   - area: carve, area:chart 등: 어느 영역의 문제인지 빠르게 좁히기 위한 라벨(모듈 이름으로 설정)
4. **자동화 대상 표시**
   - go:fix: Codex Automation이 가져가도 되는 이슈
   - go:test: test개선 PR 대상
   - go:dox: 문서 업데이트 PR 대상

이렇게 정리해두면 자동화는 아래 조건으로 오늘 밤 처리할 이슈를 단순하게 고를 수 있다.

- Codex Automation에 작업 요청 - label:go:fix + label:status:ready + label:prio:p0/p1/p2
- 진행 중인 이슈는 제외: -label:status:in-progress

+) 에이전트 코딩과 모듈화
라벨을 정리하면서 모듈화의 생각하지 못한 이점도 다시 확인할 수 있었다.

기능이 모듈 단위로 분리하고, label의 area를 모듈이름으로 달아두니 `area:*`로 범위를 지정할 때 어디를 고쳐야 하는지가 더 명확해졌다.

결합도가 높은 코드에서는 작은 수정도 전역에 영향을 주기 쉬운데, 모듈화는 의존성/컴파일 경계를 강제해서 변경 범위를 자연스럽게 좁혀졌다.

AI가 특정 기능 구현은 기가 막히게 잘하지만, 레포 전체의 암묵적 의존성과 맥락을 완전히 이해하기는 어려운 한계가 있다고 생각했었다.

그래서 책임이 명확히 분리된 모듈 구조일수록, 에이전트가 안전하게 작업할 수 있는 단위가 작아지고 결과도 더 안정적이었다.

---

## Nightly Report 워크플로

Nightly Report는 다음과 같이 동작한다 .

1. 앱에서 Firebase Analytics(GA4) 이벤트 기록
2. GA4 이벤트가 BigQuery로 Export 되어 적재
3. GitHub Actions가 매일밤 스케줄로 Python 스크립트 실행(BigQuery 조회, 최근 데이터 쿼리, Markdown로 렌더링)
4. 결과를 Markdown으로 만들고, 지정한 GitHub Issue 본문에 날짜 섹션으로 누적 기록

실제 연결은 다음글에서 ...
