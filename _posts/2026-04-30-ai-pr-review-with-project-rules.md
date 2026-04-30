---
title: "프리커밋 AI 리뷰가 놓친 것들 — 프로젝트 규칙을 학습하는 PR 리뷰봇 만들기"
description: "프리커밋 단계의 좁은 시야를 보완하기 위해 GitHub Actions 위에 PR 단위 AI 리뷰를 구축하고, 레포의 CLAUDE.md와 .claude/rules/*.md를 강제로 참조하도록 만들어 프로젝트 컨벤션 위반까지 잡아내는 리뷰봇을 만든 과정."
date: 2026-04-30
categories:
  - Development
  - AI
tags:
  - Claude Code
  - GitHub Actions
  - PR Review
  - Code Review
  - AI
  - Workflow Automation
  - Anthropic
  - DevOps
last_modified_at: 2026-04-30
howto:
  name: "프로젝트 규칙을 학습하는 PR 리뷰봇 만들기"
  time: "PT1H"
  steps:
    - name: "프로젝트 규칙 파일 정리"
      text: "레포 루트에 CLAUDE.md(코어 규칙)와 .claude/rules/*.md(보조 규칙: design-token, karpathy, commit 등)를 두어 코드와 함께 버전관리되도록 구성합니다."
    - name: "GitHub Actions 워크플로우 작성"
      text: ".github/workflows/claude-review.yml을 만들고 anthropics/claude-code-action@v1을 사용합니다. pull_request 이벤트의 opened/reopened/synchronize에 트리거되도록 설정합니다."
    - name: "권한 및 동시성 옵션 설정"
      text: "permissions에 contents:read, pull-requests:write, issues:write, id-token:write를 부여하고 concurrency.cancel-in-progress로 중복 실행을 막습니다. paths-ignore로 lock 파일과 자동 생성 코드를 제외합니다."
    - name: "프롬프트에 룰 파일 참조 강제"
      text: "프롬프트 첫 부분에서 'CLAUDE.md와 .claude/rules/*.md를 반드시 먼저 읽고 그 기준으로 리뷰하라'고 명시합니다. 모델은 워크스페이스 파일에 접근할 수 있지만 시키지 않으면 읽지 않습니다."
    - name: "리뷰 관점 우선순위 정렬"
      text: "버그 > 접근성 > 프로젝트 규칙 > 안티패턴 > 보안 > 성능 순으로 우선순위를 명시합니다. 도메인에 따라 순서가 달라야 합니다."
    - name: "작성 가드로 노이즈 제거"
      text: "ESLint/Prettier가 잡는 영역, 칭찬 코멘트, 주관적 취향, 추측성 단정을 명시적으로 금지합니다. 마지막 종합 코멘트로 머지 가능 여부를 판정하게 합니다."
    - name: "track_progress와 show_full_output 활성화"
      text: "track_progress:true로 PR에 sticky 진행 코멘트가 자동으로 달리게 하고, show_full_output:true로 디버깅 시 모델 응답 마스킹을 풀어둡니다."
faq:
  - question: "프리커밋 AI 리뷰가 있는데 왜 PR 단위로 또 돌리나요?"
    answer: "프리커밋 리뷰는 한 번에 한 파일 또는 좁은 변경 범위만 봅니다. 전체 PR을 가로지르는 일관성, 즉 '여기는 이렇게 했는데 저기는 왜 다르게 했는지'를 잡으려면 PR 단위의 넓은 시야가 필요합니다. 또 프리커밋은 빈도가 높아 컨텍스트를 풍부하게 주기 어렵지만, PR은 빈도가 낮아 룰을 통째로 주입할 수 있는 비대칭이 있습니다."
  - question: "리뷰봇이 프로젝트 컨벤션을 어떻게 알게 만드나요?"
    answer: "레포 루트의 CLAUDE.md와 .claude/rules/*.md 파일을 프롬프트에서 '반드시 먼저 읽고 그 기준으로 리뷰하라'고 명시하면 됩니다. 룰 파일이 코드와 함께 버전관리되므로 룰이 바뀌면 리뷰 기준도 자동으로 바뀝니다. 이게 단순 모델 능력보다 훨씬 중요한 차이를 만듭니다."
  - question: "PR 단위 AI 리뷰의 비용은 어느 정도인가요?"
    answer: "PR 한 번에 약 0.4 USD 수준입니다. 큰 PR이면 더 듭니다. paths-ignore로 lock 파일과 자동 생성 코드를 제외하고, concurrency.cancel-in-progress로 중복 실행을 막으면 비용을 줄일 수 있습니다."
  - question: "AI 리뷰가 '머지 가능'이라고 판정하면 그대로 머지해도 되나요?"
    answer: "안 됩니다. AI 리뷰는 놓친 걸 줄여주지 다 잡아주지는 못합니다. 도메인 결정이나 라이브러리 선택, 비즈니스 룰의 적절성 같은 판단은 여전히 사람의 영역입니다. AI 리뷰는 바닥선이지 천장이 아닙니다."
---

![PR단위 AI리뷰](/assets/img/posts/2026-04-30/pr-unit-ai-review.gif){: style="width:100%;" }


## TL;DR

- 커밋 전에 `code-review` 에이전트를 돌리고 있었지만, 결국 PR을 올리고 나서야 보이는 문제들이 있었다.
- GitHub Actions 위에 PR 단위 AI 리뷰를 붙이고, **레포 루트의 `CLAUDE.md`와 `.claude/rules/*.md`를 강제로 읽힌 다음** 그 기준으로 리뷰하게 만들었다.
- 결과적으로 AI가 잡아낸 첫 번째 리뷰에서 Tailwind v4 토큰 누락, Next.js Link `href` 객체 형태 일관성, 로딩·에러 분기 누락, silent failure UX — 내가 진짜 놓친 것들을 짚었다.
- 핵심은 모델이 아니라 **"무엇을 기준으로 리뷰할지"를 코드와 함께 버전관리하는 것**이었다.

---

## 문제 인식 — 프리커밋 리뷰가 놓치는 영역

내 워크플로우는 이미 LLM이 깊이 들어와 있다. 코드를 쓰면 커밋 전에 `code-review` 에이전트를 한 번 돌려서 의심스러운 부분을 본다. 이 단계는 빠르고, 작은 실수는 거의 다 잡힌다.

그런데 PR을 올리고 며칠 뒤에 다시 보면, 같은 패턴의 누락이 한 군데 더 있는 게 보인다. 예를 들면:

- `CategoryGrid`는 로딩·에러 분기를 만들어 `LoadingState`/`ErrorState`까지 깔았는데, 같은 페이지 옆 컴포넌트인 `DoctorCarousel`은 빈 배열만 보여주고 끝.
- `DepartmentCard`의 `<Link href>`는 객체 형태로 바꿨는데, `DoctorCard`의 `<Link>`는 그대로 템플릿 리터럴.
- Tailwind v4에서 `grid-cols-14`는 기본 스케일에 없는데 그대로 작성. dev 서버에서 데스크톱만 보면 안 보이는 케이스.

원인은 모델이 약해서가 아니다. **프리커밋 리뷰는 한 번에 한 파일 또는 좁은 변경 범위만 보기 때문**이다. 전체 PR을 가로지르는 일관성, 즉 "여기는 이렇게 했는데 저기는 왜 다르게 했지"를 잡으려면 PR 단위로 한 번 더 봐주는 게 필요하다.

거기에 더해 — 프리커밋 리뷰는 빠르게 끝나야 하니 컨텍스트가 짧다. 프로젝트 룰을 통째로 주입하면 매 커밋마다 비싸진다. PR 단계는 빈도가 낮으니 룰을 풍부하게 줘도 된다는 비대칭이 있다.

![프리커밋이 놓친것](/assets/img/posts/2026-04-30/precommit-missed.png)

## 해결 방향 — 룰을 코드와 함께 관리하는 리뷰봇

내가 원했던 건 두 가지다.

1. **PR이 열리면 자동으로 리뷰가 달릴 것** — 사람이 트리거하지 않아도 됨.
2. **그 리뷰는 내 프로젝트 규칙을 알고 있을 것** — "이 프로젝트는 `<span>` 대신 `<Text>`를 쓴다"를 매번 알려주는 게 아니라, 룰 파일을 읽어서 그 기준으로 리뷰하게 해야 함.

두 번째가 핵심이었다. 일반적인 리뷰봇은 "버그 잡아라"까지는 잘 하는데, "FSD 레이어 위반"이나 "이 프로젝트에서는 `<button>` 대신 shadcn `Button`을 쓰고, 색상은 매직넘버 hex 대신 `@theme inline` 토큰을 통해야 한다" 같은 컨벤션 위반은 잡아내지 못한다. 모델이 모르는 게 아니라 — 내 레포의 컨벤션을 모를 뿐이다.

해결책은 단순했다. **이미 코드와 함께 버전관리되고 있는 규칙 파일들을 프롬프트에서 강제로 참조하게 만든다.**

내 레포는 운 좋게도 이미 룰이 정리돼 있었다.

```
CLAUDE.md                       ← 프로젝트 코어 규칙 (네이티브 HTML 금지, FSD 레이어, A11y 필수)
.claude/rules/karpathy.md       ← 단순성, 외과적 변경 원칙
.claude/rules/design-token.md   ← HEX 하드코딩·매직넘버 금지
.claude/rules/commit.md         ← 커밋 메시지 컨벤션
```

이걸 그냥 프롬프트 안에서 "이 파일들을 먼저 읽고, 그 기준으로 리뷰해라"고 명시하면 된다.

![룰파일과 코드가 함께 진화](/assets/img/posts/2026-04-30/rules-evolve-with-code.png)

## 구현 — GitHub Actions + Claude Code Action

`anthropics/claude-code-action@v1`을 쓴다. 한 파일이면 끝난다.

`.github/workflows/claude-review.yml`:

```yaml
name: Claude PR Review

on:
  pull_request:
    types: [opened, reopened, synchronize]
    paths-ignore:
      - 'pnpm-lock.yaml'
      - 'tsconfig.tsbuildinfo'
      - 'src/shared/api/generated/**'
      - 'public/**'
      - '.next/**'
      - '.husky/**'

concurrency:
  group: claude-review-${{ github.event.pull_request.number }}
  cancel-in-progress: true

permissions:
  contents: read
  pull-requests: write
  issues: write
  id-token: write

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
        with:
          fetch-depth: 0

      - name: Claude Code Review
        uses: anthropics/claude-code-action@v1
        with:
          claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
          track_progress: true
          show_full_output: true
          prompt: |
            # 프롬프트 (아래 절에서 따로 설명)
```

몇 가지 짚을 점.

- **`paths-ignore`**: lock 파일, 빌드 산출물, 자동 생성 코드, 정적 자원. 이런 데서 리뷰가 돌면 비용만 나가고 의미가 없다.
- **`concurrency` + `cancel-in-progress`**: 같은 PR에 push가 빠르게 연달아 들어오면 이전 리뷰는 취소. 마지막 커밋만 본다.
- **`id-token: write`**: OIDC 토큰 교환에 필요. 이거 빼먹고 한참 헤맸다.
- **`track_progress: true`**: 이게 빠지면 워크플로우는 success인데 PR에는 코멘트가 안 달리는 함정에 빠진다. 모델이 "결과를 어디에 적어야 할지"를 명확히 알게 해주는 옵션. 옵션을 켜면 PR에 진행 상태 sticky comment가 자동으로 달려서, 어느 단계까지 진행됐는지 실시간으로 보인다.
- **`show_full_output: true`**: 디버깅용. 안 켜면 모델 응답이 로그에서 마스킹돼서 왜 코멘트가 안 달렸는지 추적이 안 된다.

![ 진행중인 PR 클로드 코드 액션](/assets/img/posts/2026-04-30/claude-code-action-pr.png)

### 프롬프트 설계 — 룰 주입 + 우선순위 + 작성 가드

프롬프트는 세 덩어리로 짰다.

**1. 컨텍스트 (룰 파일 주입)**

```
**프로젝트 컨텍스트**
- 핵심 규칙은 레포 루트의 `CLAUDE.md`에 정리되어 있음. 반드시 먼저 읽고 그 기준으로 리뷰할 것
- 보조 규칙:
  - `.claude/rules/karpathy.md` — 단순성, 외과적 변경, 목표 검증
  - `.claude/rules/design-token.md` — 컬러/스페이싱 토큰 (HEX 하드코딩·매직넘버 금지)
  - `.claude/rules/commit.md` — 커밋 메시지 컨벤션
```

핵심은 **"읽어라"고 명시하는 것**이다. 모델은 워크스페이스 파일에 접근할 수 있지만, 시키지 않으면 안 읽는다. 그리고 "이 룰이 우선순위가 가장 높다"는 신호도 같이 준다.

**2. 리뷰 관점 (우선순위 정렬)**

```
**리뷰 관점 (우선순위 순)**
1. 잠재적 버그/런타임 이슈 — null·undefined 안전성, race condition, useEffect 의존성, hydration mismatch...
2. 접근성(A11y) 위반 — 네이티브 <button>/<input> 사용, Label 미연결...
3. 프로젝트 규칙 위반 — <span> 대신 <Text>, FSD 레이어 위반, 매직넘버·hex 하드코딩...
4. React 19 안티패턴 — forwardRef 사용, 불필요한 useEffect...
5. 보안 — XSS, 환경변수 클라이언트 노출...
6. 성능 — 불필요한 client component, 큰 번들 import...
```

이건 **버그 > 접근성 > 프로젝트 규칙 > 안티패턴 > 보안 > 성능** 순. 내 도메인(SSR 헬스케어 페이지)에서 보안 이슈 빈도가 낮고, 버그·a11y가 가장 자주 사고 나는 영역이라 이렇게 정렬했다. 다른 도메인이면 순서가 달라야 한다.

**3. 작성 가드 (노이즈 제거)**

이게 사실 가장 시간 많이 쓴 부분이다. 가드 없이 돌리면 AI가 너무 많이 짖는다.

```
**리뷰 작성 규칙 (중요)**
- 모든 코멘트는 "왜 문제인지 + 구체적인 수정 방안"을 함께 제시. 단순 지적 금지
- 가능하면 GitHub Suggestion 블록(```suggestion)으로 바로 적용 가능한 패치 제공
- ESLint/Prettier가 잡는 스타일·포맷·세미콜론·import 순서는 코멘트 금지
- 주관적 취향("이게 더 깔끔할 듯") 금지. 규칙 위반이거나 명확한 리스크일 때만
- 확신이 없거나 추측이면 "확인 필요"로 명시하고 단정하지 말 것
- 영향이 작거나 애매하면 차라리 코멘트 안 하는 쪽 선택
- 잘 짠 부분에 칭찬 코멘트는 생략
```

핵심은 **"안 짖어도 되는 것"을 명시적으로 빼는 것**이다.

- 린트가 잡는 영역은 빼라 — husky/lint-staged가 이미 강제하니까 중복.
- 칭찬 코멘트 빼라 — 노이즈.
- 추측이면 단정 짓지 말고 "확인 필요"로 표기 — false positive를 인지 가능한 false positive로 바꾼다.

**4. 종합 코멘트 포맷 강제**

```
**마지막 종합 코멘트**
맨 마지막에 PR 전체 요약 1개:
- 변경 요약: 1-3줄
- 머지 전 반드시 봐야 할 이슈: Critical/High만 (없으면 "없음")
- 선택적 개선: Medium/Low (없으면 생략)
- 결론: 머지 가능 / 수정 필요 / 추가 논의 필요
```

이게 의외로 중요했다. 인라인 코멘트만 50개 달려 있으면 PR 작성자는 "이 PR을 머지해도 되는가"를 판단할 수 없다. 마지막에 한 문단으로 **결론**을 내려달라고 강제하면, 그게 곧 머지 게이트가 된다.

## 적용 결과 — AI가 진짜로 짚어낸 것들

첫 PR을 올렸다. 내가 만든 페이지 약 70개 파일, +1700/-90 변경. 리뷰 결과 인라인 코멘트 6건 + 종합 코멘트 1건이 달렸다.

![첫 번째 리뷰 코멘트](/assets/img/posts/2026-04-30/first-review-comment.png)
>
> ※ 두 번째로 캡쳐한 게 다른 화면이면 알려주세요 — 적용 결과 섹션 다른 자리(예: "리뷰가 한 번에 끝나지 않았다" 단락) 직후로 옮길 수 있습니다.

**Critical/High로 짚힌 것들 (3건 — 모두 진짜 버그)**

1. `sm:grid-cols-14` 사용 — Tailwind v4 기본 그리드 스케일은 1~12까지만 정의됨. globals.css의 `@theme inline`에 `--grid-template-columns-14`를 추가해야 한다고 정확히 짚음. design-token.md를 읽었기 때문에 "임의값 대신 토큰화"를 권장하는 것까지 룰에 맞춰 제안.

2. `CategoryGrid`/`FeaturedArticleList`에 로딩·에러 분기 누락 — 옆 컴포넌트(`DepartmentResultList`)는 `LoadingState`/`ErrorState`로 처리하고 있어서 "일관성이 깨진다"고 지적. **PR 전체를 가로지르는 일관성 검사가 작동했다.**

3. `DoctorDetailSearch`의 진료과 Select가 fetch 실패 시 빈 옵션만 보여주는 silent failure — "사용자가 검색 자체를 못 하는 상황이 조용히 발생한다"는 영향까지 묘사.

**Medium/Low로 짚힌 것들**

4. Radix Tabs의 `<TabsContent value={activeKind}>` 단일 패널 패턴 — idiomatic 패턴(트리거당 TabsContent 하나)에서 벗어난다는 지적. 이건 a11y trade-off가 있는 회색 지대라 한 번 토론한 뒤 idiomatic 1:1 매핑으로 바꿨다.

5. Next.js Link `href`를 템플릿 리터럴 대신 `{ pathname, query }` 객체로 — 타입 안전성과 자동 인코딩 근거 명시.

6. `useDraftValue`의 `useEffect` 동기화 플래시 가능성 — "한 프레임 동안 이전 값이 비칠 수 있다"는 미묘한 동작을 정확히 묘사. 이건 코드를 안 바꾸고 주석으로 제약을 명시하는 게 적절했다.

흥미로운 점은 **리뷰가 한 번에 끝나지 않았다**는 것. 첫 라운드에서 같은 패턴(href 객체화) 중 일부만 고치고 푸시했더니, 두 번째 라운드 리뷰에서 "DoctorCard는 누락됐다"고 정확히 짚었다. PR 전체를 보는 시야 덕분에 부분 적용도 잡힌다.

또 하나 — 리뷰어가 짚지 않은 영역도 있었다. **도메인 결정과 라이브러리 선택**. 예를 들어 의료진 카드 캐러셀에 Swiper를 도입한 부분, 또는 센터 탭에서는 한글 초성 필터를 숨기는 비즈니스 룰. AI는 코드의 정확성은 검사하지만 그게 적절한 선택인지는 판단하지 않는다. "이 의존성이 번들 크기를 감수할 만큼 가치 있나", "이 도메인 룰이 사용자 의도에 맞나" 같은 질문은 여전히 사람의 영역이다. **AI 리뷰는 바닥선이지, 천장이 아니다.**

## 한계와 트레이드오프

**비용** — PR 한 번에 약 0.4 USD. 큰 PR이면 더 든다. 작은 팀에서는 의미있는 수치다. `paths-ignore`로 노이즈를 줄이고, `concurrency.cancel-in-progress`로 중복 실행을 막은 이유.

**False positive** — 가드를 빡세게 짜도 가끔 "여긴 진짜 안 고쳐도 되는 회색지대"를 critical로 올린다. 위에서 TabsContent 패턴이 그랬다. 모든 지적을 그대로 적용하면 안 되고, **반박할 줄 알아야 한다**. 사람이 게이트를 잡아야 한다.

**워크플로우 메타 변경의 마찰** — `claude-code-action`은 보안상 PR 브랜치의 워크플로우 파일이 main과 한 글자라도 다르면 OIDC 토큰을 안 내준다. 워크플로우 옵션 하나 추가하려면 main에 먼저 푸시한 다음 PR 브랜치를 동기화해야 한다. 권한 상승 공격을 막는 정당한 가드지만, 도입 초기에는 함정이다.

**프롬프트가 코드 베이스의 일부가 된다** — 룰 파일이 바뀌면 리뷰 기준도 자동으로 바뀐다. 이건 장점이지만, 누군가 룰 파일을 잘못 수정하면 리뷰 품질이 조용히 떨어진다. 룰 파일도 PR 리뷰 대상에 들어가게 하든지, 별도 검증 단계를 두는 게 좋다.

**모델 신뢰의 한계** — "이 PR 머지해도 되나요?"에 모델이 "머지 가능"이라고 했다고 그대로 머지하면 안 된다. 내 PR 한 건은 모델이 머지 가능 판정을 했는데, 직접 살펴보니 한 컴포넌트의 라이프사이클 누락이 있었다. AI 리뷰는 **놓친 걸 줄여주지, 다 잡아주지는 않는다**.

## 마무리

핵심은 모델 선택이 아니라 **"무엇을 기준으로 리뷰하는지"를 명시적으로 코드와 함께 두는 것**이었다.

- 룰을 별도 시스템에 두지 말고 레포 안에 두자. 코드처럼 버전관리되고, PR로 변경되고, 함께 진화한다.
- 프리커밋 리뷰는 좁은 시야 + 빠른 피드백, PR 리뷰는 넓은 시야 + 풍부한 컨텍스트. 둘은 대체재가 아니라 보완재다.
- 가드(작성 규칙, 우선순위, paths-ignore)는 모델 능력이 좋아질수록 더 중요해진다. 모델이 똑똑할수록 안 짖어야 할 곳에서도 자세히 짖기 때문에.

다음에 해보고 싶은 건 두 가지. 첫째, 룰 파일 자체에 대한 메타 리뷰 — 룰 파일이 PR로 수정되면 별도 검증 워크플로우. 둘째, 리뷰 결과를 누적해서 자주 지적되는 패턴을 룰 파일에 역으로 반영하는 루프. AI 리뷰가 단발 도구가 아니라 **컨벤션을 진화시키는 피드백 시스템**이 되는 지점이다.
