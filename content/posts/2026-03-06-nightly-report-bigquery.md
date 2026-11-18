+++
title = "[AI] GitHub Actions + BigQuery로 Nightly Report 자동 수집 파이프라인 구축하기"
date = 2026-03-06T16:54:38+09:00
draft = false
tags = ["ai", "analytics", "automation", "chatgpt", "codex", "firebase", "github", "ios", "에이전트코딩", "자동화"]
categories = ["AI"]
tistory = "https://leetaek.tistory.com/88"
+++

이번 글은 “Codex Automation”을 활용해 밤새 버그를 수집·분석하고 수정 제안까지 연결하는 파이프라인을 구축한 두 번째 포스트다.

1. [[AI]  자동화를 위한 GitHub Issue 운영 체계 만들기 (라벨/템플릿/분류)](/posts/2026-02-27-codex-github-issue-ops/)
2. **[AI] GitHub Actions + BigQuery로 Nightly Report 자동 수집 파이프라인 구축하기** (작성중 ...)
3. [AI] Codex Automation으로 “이슈 1개 → Draft PR” 밤새 돌려보기 (worktree/sandbox/gh/prompt)

(현재 작성중인 미완성 글입니다....)

---

## Nightly Report 워크플로

Nightly Report는 다음과 같이 동작한다 .

1. 앱에서 Firebase Analytics(GA4) 이벤트 기록
2. BigQuery로 설정(GA4 데이터 Export, GitHub Action 조회를 위한 인증/권한 설정)
3. GitHub Actions가 매일밤 스케줄로 Python 스크립트 실행(BigQuery 조회, 최근 데이터 쿼리, markdown로 렌더링)
4. 결과를 Markdown으로 만들고, 지정한 GitHub Issue 본문에 날짜 섹션으로 누적 기록

---

### Firebase Analytics 이벤트 기록

먼저 앱에서 Firebase Analytics로 이벤트를 찍어야한다.

사실 그동안 화면 사용 시간에 대한 집계와 crash 로그만 집계하고 있었는데, 이 참에 사용성에 지장이 있을것 같은 포인트에 error_shown 이벤트를 로그를 찍어두었다.

즉, “버그를 전부 수집”하기보다 가장 먼저 잡아야 할 에러 신호부터 모으는 구조로 출발했다.

다음 단계는 이 이벤트가 **GA4 → BigQuery**로 실제 적재되는지 확인하고, GitHub actions의 스크립트에서 조회 가능한 형태로 만드는 것이다.

---

### BigQuery - GA4 데이터 Export 설정

BigQuery는 이번 파이프라인에서 GA4 이벤트가 쌓이는 저장소 역할을 한다.

Nightly Report는 이 데이터를 최근 24시간 기준으로 집계해서 GitHub Issue에 남기므로, 먼저 GA4 → BigQuery Export 연결이 필요하다.

![GCP 콘솔 화면](/images/tistory/88/01.png)

*GCP 콘솔 화면*

**준비물**

- Firebase 프로젝트에 **Google Analytics(GA4)** 가 연결되어 있어야 함
- BigQuery를 붙일 **Google Cloud 프로젝트**가 있어야 함(권한 필요)

**1) GA4 → BigQuery Export 연결**

- **목적**: Firebase Analytics(GA4) 이벤트를 BigQuery dataset에 자동 적재
- **위치**: Firebase 콘솔 → **프로젝트 설정 → 통합(Integrations) → BigQuery**

여기서 BigQuery 연결을 설정하면, 연결된 GCP 프로젝트에 GA4 Export용 dataset이 생성된다.

상세한 GA4 → BigQuery Export 연결 방법은 [문서 참고](https://firebase.google.com/docs/ab-testing/bigquery?utm_source=chatgpt.com&hl=ko#enable-bigquery-export)

- [BigQuery로 A/B 테스팅 데이터 검사  |  Firebase A/B Testing](https://firebase.google.com/docs/ab-testing/bigquery?utm_source=chatgpt.com&hl=ko#enable-bigquery-export) — firebase.google.com

**2) Export가 정상인지 확인**

- **목적**: “연결이 됐는지”를 BigQuery에서 바로 확인
- **위치**: Google Cloud 콘솔 → **BigQuery → Explorer**

정상이라면 아래가 보인다.

- dataset: analytics_XXXXXXX 형태의 dataset 생성
- 테이블: events_YYYYMMDD 테이블 생성

환경에 따라 당일 데이터는 events_intraday_YYYYMMDD로 먼저 쌓이고, 이후 events_YYYYMMDD로 확정되는 경우도 있다.

나는 이 단계에서 dataset/테이블 생성만 먼저 확인했고, error_shown 이벤트는 아직 실제 데이터가 없어서 Nightly report에 “No events found”로 찍혔다. 즉, Export는 정상이고 이벤트가 없었던 상태였다.

![](/images/tistory/88/02.png)

이제 앱에서 수집된 GA4 데이터를 BigQuery로 연결하는 과정은 끝났다.

다음은 GitHub에서 조회할 수 있도록 인증/권한을 설정해야 한다.

### BigQuery - GitHub Actions 조회를 위한 인증/권한 설정

GitHub Actions는 BigQuery를 직접 읽지 않고, **Service Account를 ‘대리(impersonate)’해서** 읽는다. 그래서 “대리할 수 있는 권한”과 “BigQuery 읽기 권한” 두 종류가 필요하다.

**준비물**

- Service Account 1개(예: nightly-report@...)
- Workload Identity Federation(Pool + Provider)

**1) Service Account 만들기**

- 목적: BigQuery를 조회할 실제 주체(서비스 계정) 생성
- 위치: [GCP 콘솔](https://console.cloud.google.com/welcome) → IAM & Admin (IAM 및 관리자)→ Service Accounts(서비스 계정)

**2) Workload Identity Pool + Provider 만들기**

- 목적: GitHub OIDC 토큰을 신뢰하고 내 레포만에서 실행된 워크플로우만 허용
- 위치: [GCP 콘솔](https://console.cloud.google.com/welcome) → IAM & Admin(IAM 및 관리자) → Workload Identity Federation(워크로드 아이덴티티 제휴)
- 역할:
  - Pool: 외부 인증(여기선 GitHub OIDC)을 관리하기 위해 담아두는 컨테이너.
  - Provider: 실제로 “GitHub OIDC를 신뢰한다”고 설정하는 곳. pool 안에서 provider를 만들고, repository == LeeTaek/Carve 조건(내 경우)을 걸어 허용 범위를 레포 단위로 제한한다.

**3) Service Account(서비스 계정)에 대리 권한 부여**

- 목적: GitHub Actions를 대리할 서비스 계정 설정
- 위치: Service Accounts(서비스 계정) → (대상 SA) → Permissions(권한) → 서비스 계정 권한 관리 → 엑세스 관리 → principalSet(레포 제한) 추가 및 권한 부여
- 부여할 권한과 역할:
  - Workload Identity User(작업 부하 ID 사용자)
  - Service Account Token Creator(서비스 계정 토큰 사용자): 요게 빠지면 Codex 스크립트 실행시 iam.serviceAccounts.getAccessToken 403으로 막힌다.

**4) Service Account(서비스 계정)에 BigQuery 권한 부여**

- 목적: 서비스 계정이 BigQuery를 조회할 수 있는 권한 부여
- 위치:
  - [GCP 콘솔](https://console.cloud.google.com/welcome) → IAM & Admin(IAM 및 관리자) → IAM → 서비스 계정 → 역할 지정
- 역할:
  - BigQuery Job User: 쿼리 실행할 수 있는 권한
  - BigQuery Data Viewer: GA4 Export dataset을 읽을 수 있는 권한

---

## GitHub 연결

BigQuery 쪽 설정(Export + 인증/권한)이 끝났다면, 이제 GitHub Actions가 그 설정값들을 참조할 수 있게 GitHub 레포에 값을 등록해야 한다.

필요한 것은 다음과 같다.

- GitHub Actions가 사용할 **Workload Identity Provider 경로**
- GitHub Actions가 대리할 **Service Account 이메일**
- GitHub Actions에서 조회/리포트 할 **Dataset 번호**

이 값들은 워크플로(yml)에 직접 하드코딩할 수도 있지만, GitHub Settings에 저장해두는 편이 안전하고 편하다.

**준비물**

**1) Workload Identity Provider 경로**

- 예:
- projects/<Project_Number(숫자)>/locations/global/workloadIdentityPools/<Pool_ID>/providers/<provider_ID>
- 위치: [GCP 콘솔](https://console.cloud.google.com/welcome) → IAM & Admin(IAM 및 관리자)→ Workload Identity Federation(워크로드 아이덴티티 제휴) → PoolID 복사 → ProviderID 복사

**2) Service Account 이메일**

- 위치: GCP 콘솔 → IAM & Admin(IAM 및 관리자) → Service Accounts(서비스 계정) → 이메일 복사

**3) BQ_DATASET**

- GitHubAction에서 조회할 DataSet 번호
- 위치: GCP 콘솔 → BigQuery → 프로젝트 → analytics_xxxxxxxxx 복사

**4) NIGHTLY_ISSUE_NUMBER**

- Nightly report 이슈를 누적 기록할 GitHub Issue 번호.
- 위치: GitHub → Issues → Nightly report 이슈 열기 → URL에서 번호 확인
  - 예: .../issues/2 이면 NIGHTLY_ISSUE_NUMBER=2
- 없다면 Issue에 만들어서 등록

### GitHub Settings에 값 등록하기

준비물(Provider 경로, Service Account 이메일, BQ_DATASET, NIGHTLY_ISSUE_NUMBER)을 확인했다면, 이제 GitHub 레포에 값을 등록한다.

- 위치: GitHub 레포 → **Settings → Secrets and variables → Actions**

**Secrets(인증 관련)**

- WORKLOAD_IDENTITY_PROVIDER
- GCP_SERVICE_ACCOUNT

**Variables(스크립트 설정값)**

- BQ_DATASET
- NIGHTLY_ISSUE_NUMBER

### 테스트: Nightly Report를 수동으로 한 번 돌려보기

- 목적: “GCP 인증/권한 + GitHub Secrets/Variables + 스크립트 동작”을 한 번에 확인
- 방법: GitHub Actions에서 해당 워크플로를 **Run workflow**로 수동 실행
- 기대 결과:
  1. Actions 로그에서 BigQuery 쿼리 수행 로그가 찍힘
  2. 지정한 Issue 본문에 YYYY-MM-DD 섹션으로 결과가 append 됨
  3. 이벤트가 없으면 No events found가 찍혀도 정상(파이프라인은 통과한 것)

![](/images/tistory/88/03.png)

![](/images/tistory/88/04.png)

정상 동작한다.
