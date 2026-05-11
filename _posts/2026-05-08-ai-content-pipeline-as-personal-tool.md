---
title: "AI 콘텐츠 파이프라인을 1인 도구로 다시 짜면서 바뀐 결정 4개"
description: "매일 뉴스 카드를 Threads에 자동 발행하는 시스템을 SaaS 기본값으로 짓다가 1인 도구로 다시 짠 과정. 운영 UI 전체 차단, 사람이 승인한 카드만 발행, 부분 실패 허용, Cron이 호출하는 API만 인증 — 사용자 한 명·인터페이스는 텔레그램이라는 전제에서 따라온 결정 4개를 정리한 글."
date: 2026-05-08
categories:
  - Development
  - AI
tags:
  - Next.js
  - Vercel Cron
  - Telegram Bot
  - AI
  - Threads
  - Personal Tool
  - Automation
  - Workflow Automation
  - Tool Use
  - Anthropic
last_modified_at: 2026-05-08
howto:
  name: "AI 콘텐츠 파이프라인을 1인 도구로 짓기"
  time: "PT2H"
  steps:
    - name: "사용자와 인터페이스 정의"
      text: "사용자가 한 명임을 인정하고 검수 인터페이스를 화면이 아니라 메시징(텔레그램)으로 잡습니다. 이 한 줄이 나머지 결정의 기준이 됩니다."
    - name: "운영 환경 UI 라우트 전체 차단"
      text: "Next.js 미들웨어(proxy.ts)에서 NODE_ENV === 'production'일 때 모든 UI 라우트에 404를 반환합니다. matcher에서 /api를 제외해 Cron과 텔레그램 웹훅은 영향을 받지 않게 합니다."
    - name: "상태 머신 강제"
      text: "시트의 상태값을 draft → reviewed → publishing → published 순서로 단방향 전이시키고, 발행 Cron은 reviewed 상태인 카드만 가져가게 합니다. AI 출력이 곧장 published로 가는 경로는 만들지 않습니다."
    - name: "텔레그램 인라인 버튼으로 검수 UI 대체"
      text: "수집된 카드를 텔레그램으로 보내고 메시지에 '승인 / 즉시발행 / 건너뛰기' 인라인 버튼을 붙입니다. 발행 권한이 메시징 안에만 남게 됩니다."
    - name: "stale 만료 정책 도입"
      text: "REVIEWED_STALE_MS(예: 6시간)를 정의하고, 발행 Cron이 발행 작업보다 먼저 expireStaleReviewedAction()을 실행해 시간 지난 reviewed 카드를 만료시킵니다."
    - name: "부분 실패 허용"
      text: "AI Tool Use로 받은 카드 배치를 filterValidCards로 검증해 분량 등 룰을 어긴 카드만 빼고 나머지는 발행 후보로 통과시킵니다. 응답 전체를 throw 하지 않습니다."
    - name: "Cron API 인증만 살리기"
      text: "verifyCronAuth에서 Authorization: Bearer ${CRON_SECRET} 헤더를 검사합니다. Vercel Cron이 자동으로 헤더를 붙여 주고, 텔레그램 웹훅은 별도 시크릿 토큰으로 보호합니다. 그 외 사용자 인증은 만들지 않습니다."
faq:
  - question: "어드민 UI 없이 텔레그램으로만 검수해도 충분한가요?"
    answer: "1인 도구라면 충분합니다. 검수자가 한 명이고 검수 시간이 카드당 10초 안쪽이라면, 폰에서 메시지를 읽고 인라인 버튼을 누르는 게 노트북을 켜고 어드민 페이지에 로그인하는 것보다 짧습니다. 다만 카드를 옆에 두고 비교하거나 한 번에 100개씩 일괄 작업해야 한다면 어드민 UI가 맞습니다."
  - question: "운영 환경에서 UI 라우트를 통째로 닫아도 괜찮나요?"
    answer: "외부에서 접근할 이유가 없는 경우에 한해 괜찮습니다. UI 라우트가 운영에 없으면 인증 흐름, CSP, 세션 관리 같은 부가 코드와 그에 따라오는 공격면이 통째로 사라집니다. Next.js 미들웨어에서 NODE_ENV === 'production'일 때 NextResponse 404를 반환하면 한 줄로 처리됩니다."
  - question: "AI가 만든 카드를 곧장 발행하지 않는 이유는?"
    answer: "사람의 손을 거치지 않은 카드가 인터넷에 나갈 경로를 시스템에서 없애기 위해서입니다. AI가 톤을 잘못 잡거나 시트가 꼬여도, 텔레그램에서 승인 버튼을 누르지 않은 카드는 reviewed 상태가 되지 못하고 Cron 발행기는 reviewed만 봅니다. 발행 권한이 메시징 안에 갇히게 됩니다."
  - question: "부분 실패 시 자동 재생성을 넣지 않은 이유는?"
    answer: "1인 도구에서는 자동 재생성의 비용 대비 효용이 낮다고 봤습니다. Cron이 일정 주기로 새 배치를 가져오니 재생성이 필요하면 그냥 건너뛰고 다음 배치를 기다리면 됩니다. 검수자가 텔레그램에서 한 번 더 손을 대는 비용이 자동 보정 로직을 만들고 유지하는 비용보다 쌉니다."
  - question: "이 설계를 SaaS(다인 운영)로 확장할 수 있나요?"
    answer: "가능은 하지만 운영 UI 전체를 새로 만들어야 합니다. 이 설계는 '사용자는 한 명, 인터페이스는 텔레그램'이라는 전제 위에 4개 결정이 일관되게 따라온 구조이고, 그 전제가 바뀌면 결정도 다시 내려야 합니다. SaaS로 진화시킬 가능성이 있다면 처음부터 어드민 UI를 두고 가는 편이 재설계 비용 측면에서 더 쌉니다."
---

![텔레그램 검수 메시지 — 이 시스템의 진짜 UI](/assets/img/posts/2026-05-08/telegram-approval-hero_1.png)

## TL;DR

- 매일 뉴스 카드를 만들어 Threads에 자동 발행하는 시스템을 짰다. 사용자는 한 명, 나다.
- 처음엔 SaaS 짓듯이 시작해서 멀티 유저·어드민 UI·권한 매트릭스를 그리고 있었다. 다시 시작하면서 한 명짜리 도구로 좁혔다.
- 결국 결정 네 개가 남았다: 운영 UI 전체 차단, 사람이 승인한 카드만 발행, 부분 실패 허용, Cron이 호출하는 API만 인증.
- 네 결정 모두 "인터페이스는 텔레그램이고 사용자는 한 명"이라는 전제에서 따라온 결과로 보였다.

## 문제 인식

처음엔 평범하게 시작했다. Next.js 깔고, 검수용 어드민 페이지를 짰다. 카드 리스트 보여 주고, 승인 버튼 만들고, 발행 결과 모니터링하는 대시보드를 그렸다.

이틀쯤 써보니 몇 가지 어색한 점이 보였다.

- 어드민 페이지를 켜놓고 일하지 않았다. 검수는 폰에서 했다. 출퇴근길, 점심시간.
- 승인 한 번에 5초 걸렸다. 페이지 열고 → 카드 읽고 → 버튼 누르는 행위에 노트북이 굳이 필요하지 않았다.
- 운영에 올린 검수 페이지는 외부에서 접근할 일이 없는데도 인증·CSP·로그인 흐름을 만들어야 했다. 공격면만 늘었다.
- "내가 자리에 없을 때 자동으로 발행"이라는 옵션을 만들었다가, 내가 직접 짠 파이프라인이 내 의도와 무관한 글을 인터넷에 흘리는 일을 한 번 겪었다. 톤이 망가진 카드 하나가 그대로 나갔다.

기능을 더 붙여서 풀릴 문제는 아니었던 것 같다. **사용자 한 명짜리 도구를 SaaS 만들듯이 짓고 있었다.** SaaS는 모르는 다수가 들어온다는 전제로 짓는다. 회원가입, 권한, 어드민 UI, 모니터링 대시보드. 그 전제로 짓다 보니 결정할 때마다 "누가 쓰는가"를 자꾸 일반론으로 답하게 됐다. 내가 쓰는 도구인데도 그랬다.

그래서 한 가지 질문만 남기고 다시 시작했다.

> 이 시스템의 사용자는 누구이고, 어디서 그 사용자를 만나는가?

답: 한 명, 텔레그램에서.

이 답을 기준으로 다시 보니 나머지 결정도 거기서 따라왔다.

## 해결 방향 설계

UI를 화면이 아니라 메시징 인터페이스로 본다. 웹 페이지는 개발할 때만 쓰는 디버그 도구로 두기로 했다. 운영 환경에서는 두 가지만 살아 있으면 된다.

1. Cron이 호출하는 API — 정해진 시간에 수집/발행
2. 텔레그램 봇 인터페이스 — 카드 승인/즉시발행/건너뛰기

웹 UI는 운영에서 404로 닫았다. 외부에서 접근할 이유가 없어 존재 자체를 없앤 셈이다.

여기에 "AI는 초안만, 사람은 결정만"이라는 두 번째 축이 자연스럽게 붙었다. 인터페이스가 텔레그램이면 검수는 사람의 판단을 거치게 되고, 시스템은 그 판단을 거치지 않은 카드를 받지 않으면 된다. 그래서 AI가 만든 카드는 기본 상태가 `draft`이고, 텔레그램에서 승인 버튼을 누른 카드만 `reviewed`가 된다. Cron 발행기는 `reviewed`만 본다.

이 두 축이 정해지자 부분 실패 처리도, 인증 방식도, 시간 제약도 거의 자동으로 따라왔다.

전체 흐름을 한 장으로 보면 이렇다.

![시스템 아키텍처 — 운영에서 살아있는 진입점은 Cron API와 텔레그램 봇뿐](/assets/img/posts/2026-05-08/ai-content-pipeline-as-personal-tool_2.png)

운영 환경에서 살아 있는 진입점은 Cron이 호출하는 API와 텔레그램 봇 두 개뿐이다. 그림에서 웹 UI 박스가 흐름선에 닿지 않고 따로 떠 있는 건 그래서다.

## 구현

### 결정 1 — 운영 환경에서 UI 라우트는 통째로 404

미들웨어에 한 줄을 넣어 처리했다. 매처가 이미 `/api`를 제외하고 있어 Cron과 텔레그램 웹훅은 영향이 없다.

```ts
// src/proxy.ts
export function proxy(request: NextRequest) {
  // 운영 환경에서는 UI 라우트 전체 차단.
  // matcher가 이미 /api/* 를 제외하므로 cron/webhook은 영향 없음.
  if (process.env.NODE_ENV === 'production') {
    return new NextResponse(null, { status: 404 })
  }
  return routeGuard(request) ?? applyCSP(request)
}

export const config = {
  matcher: [
    {
      source: '/((?!api|_next/static|_next/image|favicon.ico).*)',
      // ...
    },
  ],
}
```

이 짧은 코드 덕에 따라온 효과가 몇 가지 있었다.

- 검수 페이지에 인증 흐름을 만들지 않아도 됐다. 운영에 존재하지 않으니까.
- 운영 환경 CSP는 어차피 안 쓰이니 개발용만 신경 쓰면 됐다.
- 누가 도메인을 알아내서 `/admin` 같은 걸 찔러봐도 그냥 404다. 어떤 라우트가 있는지 단서가 안 남는다.

`middleware.ts`를 따로 두지 않고 `proxy.ts` 한 곳에 모은 것도 같은 맥락에서다. 운영 진입점이 한 군데에서 결정된다는 걸 코드로도 보이게 하고 싶었다.

![운영 환경에서는 UI 라우트가 통째로 404](/assets/img/posts/2026-05-08/404.png)

### 결정 2 — 사람이 승인한 카드만 발행되는 상태 머신

시트의 상태값을 `draft → reviewed → publishing → published` 순서로 거치게 했다. AI 출력이 곧장 `published`로 가는 경로는 일부러 만들지 않았다.

```ts
// src/entities/daily-news/model/constants.ts
export const SHEET_STATUS = {
  draft: 'draft',
  reviewed: 'reviewed',
  publishing: 'publishing',
  published: 'published',
  partial: 'partial',
  failed: 'failed',
  skipped: 'skipped',
} as const

/** reviewed 카드의 stale 판정 기준 (ms). 수집 후 6시간 지나면 만료 */
export const REVIEWED_STALE_MS = 6 * 60 * 60 * 1000
```

상태 전이는 단방향이고, 시스템에 정의된 분기는 다음이 전부다.

![카드 상태 머신 — AI는 draft까지, 사람 승인 후에만 reviewed](/assets/img/posts/2026-05-08/ai-content-pipeline-as-personal-tool_1.png)

발행 Cron은 `reviewed` 상태인 카드만 가져온다. `draft`는 건드리지 않는다. 결과적으로 발행 권한은 텔레그램 인라인 버튼 안에만 남는다.

```ts
// src/features/daily-news-publish/lib/telegram.actions.ts
const buttons: InlineButton[][] = [
  [
    { text: '✅ 승인', callback_data: `approve:${data.rowNumber}` },
    { text: '🚀 즉시발행', callback_data: `publish:${data.rowNumber}` },
  ],
  [{ text: '⏭ 건너뛰기', callback_data: `skip:${data.rowNumber}` }],
]
```

`REVIEWED_STALE_MS = 6시간`은 처음엔 안 보이던 항목이었다. 어제 승인한 카드가 오늘까지 살아 있으면 시의성이 안 맞는다. 텔레그램 메시지는 한 번 흘러가면 위로 올라오지 않으니, 그 위에서 승인된 카드도 같은 수명을 따라가는 게 자연스러웠다. 어드민 페이지를 들고 있을 때는 "다시 확인하면 되지"라며 미뤘는데, 메시징으로 옮기고 나서야 만료 시간을 명시적으로 박아 두기로 했다.

발행 Cron에서 만료 처리가 발행보다 먼저 일어난다.

```ts
// src/app/api/cron/publish/route.ts
const expiredCount = await expireStaleReviewedAction()
const reviewed = await getReviewedCardsAction()

const outcomes: PublishOutcome[] = []
for (const card of reviewed) {
  const result = await publishCardByRowAction(card.rowNumber)
  // ...
}
```

### 결정 3 — 한 카드 망쳐도 나머지는 살린다

AI Tool Use로 카드 5개를 한 번에 받는데, 그중 한 카드가 Threads의 500자 하드 리밋을 넘기는 일이 자주 생긴다. 처음엔 응답 전체를 throw 했더니 그 배치 카드 5장이 다 사라지는 일이 반복됐다.

```ts
// src/features/daily-news-collect/lib/collect.actions.ts
function filterValidCards(cards: DailyNews['Card'][]): DailyNews['Card'][] {
  return cards.filter((card, idx) => {
    if (card.card_type !== 'news') return true

    const headline = card.headline || `card_${idx + 1}`
    const mainLen = (card.main_text ?? '').length
    if (mainLen > MAIN_TEXT_MAX) {
      console.warn(
        `[수집] "${headline}" 카드 제외: main_text ${mainLen}자 (상한 ${MAIN_TEXT_MAX} 초과)`
      )
      return false
    }
    // replies 검증도 동일 패턴 — 위반 카드만 빠지고 나머지는 통과
    // ...
    return true
  })
}
```

규칙은 단순하다. 위반 카드는 빼고, 나머지는 그대로 발행 후보로 보낸다. 텔레그램에서 내가 받는 카드는 분량 검증을 통과한 것들만 남으니 검수에 쓰는 인지 비용도 같이 줄었다.

SaaS 짓던 머리로는 "AI 출력 품질이 떨어지면 자동 재생성"을 넣고 싶었다. 1인 도구라면 재생성이 필요할 때 텔레그램에서 그냥 건너뛰고 다음 배치를 기다리면 된다. Cron이 4시간마다 도니 어차피 곧 새 카드가 들어온다. 자동 보정 로직은 일단 넣지 않기로 했다.

### 결정 4 — 운영의 공격면은 Cron API 한 줄

운영 환경에서 외부에 노출된 엔드포인트는 사실상 `/api/cron/*`와 `/api/telegram/webhook`뿐이다. 둘 다 호출자가 정해져 있다.

```ts
// src/app/api/cron/_lib/auth.ts
export function verifyCronAuth(request: Request): NextResponse | null {
  const authHeader = request.headers.get('authorization')
  if (authHeader !== `Bearer ${env.CRON_SECRET}`) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
  }
  return null
}
```

Vercel Cron이 자동으로 `Authorization: Bearer ${CRON_SECRET}` 헤더를 붙여 주고, 텔레그램 웹훅은 별도 시크릿 토큰으로 보호한다. 그 외에 인증해야 할 사용자가 없어서 회원가입도, 비밀번호 정책도, 세션 만료도 따로 만들지 않았다.

```json
// vercel.json — KST 기준 06:00 / 10:00 / 16:00 수집, 08:00 / 12:00 / 18:00 발행
{
  "crons": [
    { "path": "/api/cron/collect", "schedule": "0 21 * * *" },
    { "path": "/api/cron/publish", "schedule": "0 23 * * *" },
    { "path": "/api/cron/collect", "schedule": "0 1 * * *" },
    { "path": "/api/cron/publish", "schedule": "0 3 * * *" },
    { "path": "/api/cron/collect", "schedule": "0 7 * * *" },
    { "path": "/api/cron/publish", "schedule": "0 9 * * *" }
  ]
}
```

수집과 발행이 2시간 간격으로 페어링되어 있다. 수집 직후 텔레그램에 카드가 도착하고, 2시간 안에 폰에서 승인을 누르면 다음 발행 Cron이 그 `reviewed` 카드를 가져간다. 2시간을 넘기면 `REVIEWED_STALE_MS`가 잡아낸다. 시간 간격은 폰으로 검수하는 데 걸리는 현실적인 여유를 기준으로 잡았다.

## 적용 결과

- 운영에서 외부에 노출된 라우트는 3개(`collect`, `publish`, `webhook`)뿐이라 보안 검토할 곳이 좁아졌다.
- 검수 어드민 페이지 코드 베이스를 들고 있지 않아서, 그쪽에서 생길 수 있는 권한·세션·CSRF 류 버그는 처음부터 안 만들었다.
- 발행되는 카드는 내가 텔레그램에서 버튼을 누른 것뿐이다. AI가 톤을 망쳐도, 시트가 꼬여도, 사람의 손을 거치지 않은 게 인터넷에 나갈 경로가 시스템에 없다.
- 검수 시간은 카드당 10초 안쪽이다. 폰에서 메시지를 읽고 버튼을 누르면 끝이라 어드민 페이지를 켤 일이 없다.
- 부분 실패가 일어나도 그날 콘텐츠가 0이 되지 않는다. 5장 중 1장이 무너지면 4장이 발행된다.

![Cron이 자동으로 발행한 메인 + 답글 체인](/assets/img/posts/2026-05-08/ai-content-pipeline-as-personal-tool_3.png)

## 한계와 트레이드오프

이 설계가 모든 상황에 맞지는 않는다.

- 검수자가 한 명이라 병목이다. 내가 자리에 없으면 그날 발행이 빈다. 다인 운영에는 맞지 않는 구조다.
- 메시징 인터페이스의 한계가 있다. 카드를 옆에 두고 비교하기 어렵고, 한 번에 일괄 작업하기도 불편하다. 100개씩 검수해야 하는 일이라면 어드민 UI가 맞을 것 같다.
- 확장하려고 할 때 재설계 비용이 든다. "친구한테도 열어줄까" 같은 생각이 떠오르면 운영 UI를 다시 만들어야 한다. 처음부터 SaaS로 진화시킬 생각이라면 어드민 UI를 두고 가는 편이 더 쌀 거다.
- 개발 환경과 운영 환경의 거리가 크다. 운영에서 안 보이는 UI를 개발에서 보며 일하기 때문에, 운영에서만 생기는 라우팅 이슈는 별도로 검증해야 한다.

이 중 어느 것도 버그라기보다는, 1인 도구로 두기로 한 결정에 따라오는 비용에 가깝다고 본다. SaaS 만들 때 드는 비용을 피한 대신 1인 도구의 비용을 내는 셈이다.

## 마무리

1인 도구를 짓는다는 건 SaaS와 같은 코드를 더 적게 쓰는 일이 아니라, 결정의 기준을 조금 다른 데에 두고 가는 일에 가까운 것 같다.

이 레포에서 내린 결정 4개 — 운영 UI 차단, reviewed 상태 강제, 부분 실패 허용, Cron API만 인증 — 는 하나씩 떼어 보면 평범한 엔지니어링 결정이다. 같이 놓고 보니 한 가지 질문에 대한 답을 따라간 결과로 보였다.

> 이 시스템의 사용자는 누구이고, 어디서 그 사용자를 만나는가?

이 질문에 먼저 답해 두지 않으면 SaaS의 기본값이 알아서 자리를 채워 버린다는 걸, 한 번 갈아엎고 나서야 알았다. 다음에 비슷한 도구를 짓는다면 이 질문부터 적어 두고 시작하려고 한다.
