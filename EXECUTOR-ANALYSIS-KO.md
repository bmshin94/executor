# Executor 분석 정리 (한국어)

> AI 에이전트용 오픈소스 통합 레이어 `Executor`를 코드베이스까지 직접 열어보고
> 분석한 내용 + 활용/수익화 아이디어 정리 문서.

## 🔗 관련 링크

| 항목 | 주소 |
|---|---|
| 원본 저장소 (upstream) | https://github.com/UsefulSoftwareCo/executor |
| 이 포크 저장소 | https://github.com/bmshin94/executor |
| 공식 웹사이트 | https://executor.sh |
| 공식 문서 | https://executor.sh/docs |
| Discord 커뮤니티 | https://discord.gg/eF29HBHwM6 |
| 코드 해설 (DeepWiki) | https://deepwiki.com/UsefulSoftwareCo/executor |
| npm 패키지 | https://www.npmjs.com/package/executor |

---

## 1. 한 줄 요약

> **"Connect any agent to everything"** — AI 에이전트를 세상의 모든 API에 연결해주는
> 중간 다리(통합 레이어).

| 항목 | 값 |
|---|---|
| 라이선스 | MIT |
| 저작권 | Copyright (c) 2026 Rhys Sullivan |
| 언어 | TypeScript (Bun + Turborepo 모노레포) |
| 워크스페이스 버전 | `1.4.0-beta.0` (베타) |
| GitHub Stars | 3,912 |
| Forks | 317 |
| 저장소 생성일 | 2026-02-07 |
| 열린 이슈 | 107 |

---

## 2. 어떤 문제를 푸는가

### 문제: N × M 설정 지옥

```
Claude Code  →  GitHub 토큰 입력, Slack 키 입력, Notion 키 입력...
Cursor       →  GitHub 토큰 또 입력, Slack 키 또 입력...
ChatGPT      →  또 입력...
```

에이전트 5개 × 도구 20개 = **100번 설정**.
같은 API 키를 도구마다 복붙해야 하고, 권한 관리도 제각각.

### 해결: 중간에 한 층을 둔다

```
                    ┌─────────────────┐
Claude Code ──┐     │                 │ ──→ GitHub API
Cursor     ───┼──→  │    EXECUTOR     │ ──→ Slack API
ChatGPT    ───┤     │  (한 개의 카탈로그) │ ──→ Notion API
내 자작 에이전트 ─┘     │  인증 + 정책 + 로그  │ ──→ 사내 API
                    └─────────────────┘
```

한 번만 등록하면 모든 에이전트가 같은 카탈로그를 공유한다. (20번만 설정)

---

## 3. 핵심 개념 4가지

| 개념 | 뜻 | 쉬운 비유 | 예시 |
|---|---|---|---|
| **Integration** | 도구들의 묶음. OpenAPI / GraphQL / MCP 서버 URL을 주면 자동으로 도구 목록 생성 | 📚 메뉴판 한 권 | "GitHub API 전체" |
| **Connection** | 통합의 인증된 인스턴스. 하나의 통합에 여러 개 가능 | 🎫 내 회원카드 | GitHub-`work`, GitHub-`personal` |
| **Policy** | 도구 단위 권한: `허용` / `승인 필요` / `차단` | 🚦 신호등 | `GET`은 허용, `DELETE`는 차단 |
| **Secret** | Executor 안에 저장하지 않음. **포인터만** 보관 | 🔐 금고 주소 쪽지 | `op://`, `keychain://`, `env://`, `file://` |

### 도구 주소 체계

```
<integration>.<scope>.<connection>.<tool>
예) github.work.main.createIssue
```

### 비유로 이해하기

- **회사 출입 관리실**: 에이전트(인턴)에게 열쇠 복사본을 주지 않고, 안내 데스크(Executor)가
  대신 다녀온다. 삭제 요청은 "사장님 승인 필요". 모든 출입 기록이 남는다.
- **레스토랑 웨이터**: 손님(에이전트)은 주방(실제 API)에 들어갈 수 없다. 메뉴판(카탈로그)을
  보고 주문만 하면 웨이터(Executor)가 처리한다.
- **만능 멀티탭**: 뒤쪽엔 온갖 API를 꽂고, 앞쪽엔 **MCP라는 표준 구멍 하나**만 낸다.

---

## 4. 폴더 구조 (실측)

### `apps/` — 사용자가 보는 제품들

| 폴더 | 정체 |
|---|---|
| `cli/` | `executor` 터미널 명령어 + 백그라운드 데몬 |
| `desktop/` | Mac/Win/Linux 네이티브 데스크톱 앱 |
| `local/` | CLI와 데스크톱이 공유하는 로컬 런타임 |
| `cloud/` | Executor Cloud (유료 호스팅 상품) |
| `host-selfhost/` | Docker 자체 호스팅 서버 |
| `host-cloudflare/` | Cloudflare Worker 배포판 |
| `marketing/` | executor.sh 홍보 사이트 |
| `docs/` | 공식 문서 |

### `packages/` — 엔진 부품들

| 폴더 | 정체 |
|---|---|
| `core/sdk` | 계약(contract), 플러그인 배선, 스코프, 정책, 시크릿 — 심장부 |
| `core/api`, `core/cli`, `core/execution` | API 서버 / CLI 코어 / 실행 엔진 |
| `kernel/runtime-quickjs`<br>`kernel/runtime-deno-subprocess`<br>`kernel/runtime-workerd-subprocess` | **코드 샌드박스 3종** — 에이전트 코드를 격리 실행 |
| `plugins/openapi`, `graphql`, `mcp`, `toolkits` | 통합 타입별 플러그인 |
| `plugins/onepassword`, `keychain`, `encrypted-secrets`,<br>`file-secrets`, `workos-vault` | **비밀키 보관소 플러그인 5종** |
| `hosts/mcp` | MCP 표면 (에이전트가 붙는 입구) |
| `react`, `app` | 웹 UI (React 19 + TanStack Router + Tailwind 4 + Vite 8) |

정리하면 **`apps` = 완성품, `packages` = 부품**.

---

## 5. 설치 및 사용법

### 실행 방식 4가지 (기능은 전부 동일, 포장만 다름)

| 방식 | 추천 대상 | 난이도 |
|---|---|---|
| Executor Cloud | 제일 빠름, 설치 없음 | ⭐ |
| Desktop 앱 | 일반 PC | ⭐⭐ |
| CLI | 서버/헤드리스 환경 | ⭐⭐ |
| Docker / Cloudflare | 자체 인프라 | ⭐⭐⭐ |

### CLI 설치 (Node.js 20 이상 필요)

```bash
npm install -g executor   # pnpm add -g / bun add -g / yarn global add 도 가능
executor install          # 재부팅해도 살아있는 백그라운드 서비스 설치
executor web              # 웹 UI → http://127.0.0.1:4788
```

잠깐만 띄우려면: `executor web --foreground`

### 에이전트 연결 (MCP)

```bash
# HTTP 방식
npx add-mcp http://127.0.0.1:4788/mcp --transport http --name executor

# stdio 방식
npx add-mcp "executor mcp" --name executor
```

> ⚠️ 대부분의 MCP 클라이언트는 **시작할 때만** 서버를 로드한다.
> 추가 후 클라이언트를 재시작하거나 새 채팅을 열어야 도구가 보인다.

### 통합 추가

웹 UI에서 `Add Integration` → OpenAPI/GraphQL/MCP URL 붙여넣기 (타입 자동 감지)

CLI에서:

```bash
executor call executor openapi addIntegration '{
  "spec": "https://petstore3.swagger.io/api/v3/openapi.json",
  "namespace": "petstore",
  "baseUrl": "https://petstore3.swagger.io/api/v3"
}'

executor tools integrations   # 확인
```

> `baseUrl`은 OpenAPI 문서의 `servers`가 상대경로(`/api/v3` 등)일 때 지정.

### 도구 사용

```bash
executor tools search "send email"        # 의도로 검색
executor call github issues --help        # 네임스페이스 탐색
executor call github issues create '{"owner":"octocat","repo":"Hello-World","title":"Hi"}'
executor resume --execution-id exec_123   # 승인 대기 중인 실행 재개
```

### 소스 개발

```bash
bun install
bun run bootstrap   # 필수! 안 하면 dev 서버가 실패함
bun run dev         # http://127.0.0.1:4788
```

> 공식 문서(`apps/docs/index.mdx`)에 **"셋업 프롬프트"** 전문이 들어있다.
> 복붙해서 Claude Code / Cursor에 주면 알아서 설치·연결까지 해준다.

---

## 6. 플러그인? 스킬? MCP?

### 정답: **MCP 서버** (정확히는 **MCP 프록시 + 통합 플랫폼**)

| 구분 | 해당? | 설명 |
|---|:---:|---|
| MCP 서버 | ✅ | `packages/hosts/mcp`가 MCP 표면 제공. 에이전트가 여기에 붙는다 |
| MCP 클라이언트 | ✅ | `plugins/mcp`로 *다른* MCP 서버를 흡수 (프록시) |
| 플러그인 | ❌ | Claude Code 플러그인이 아님. 자기 자신이 플러그인 **호스트** |
| 스킬(Skill) | ❌ | Claude Skill 아님. MCP `skills` 도구로 가이드를 노출하긴 함 |
| 라이브러리/SDK | ✅ | `@executor-js/sdk`로 코드에 직접 임베드 가능 |

```
[Claude Code] ─MCP─→ ┃          ┃ ─MCP──→ [다른 MCP 서버들]
[Cursor]      ─MCP─→ ┃ EXECUTOR ┃ ─HTTP─→ [OpenAPI API들]
[ChatGPT]     ─MCP─→ ┃          ┃ ─HTTP─→ [GraphQL 엔드포인트]
       (서버 역할) ↑              ↑ (클라이언트 역할)
```

### 설계 철학 (vision.md 인용)

> **"열린 플러그인 이음새는 오직 하나: 통합(integrations)"**
> 나머지(MCP 호스트, 워크플로, 스토리지 등)는 전부 1급 내장 기능으로 유지한다.

세상에 API는 수천 개라 통합 타입만 열어두고, 나머지는 일부러 닫아뒀다.
"모든 것을 플러그인으로 만들지 말 것"이 명시된 규율.

---

## 7. API 토큰이 필요한가?

토큰 이야기는 **두 레이어**로 나뉜다.

### 레이어 1 — Executor ↔ 실제 API (**필요함**)

`packages/core/sdk/src/http-auth/`에서 확인된 지원 인증 방식:

| 방식 | 설명 |
|---|---|
| `none` | 공개 API — 토큰 불필요 |
| `apiKey` | API 키 |
| `bearer` | Bearer 토큰 |
| `oauth2` | OAuth 2.0 로그인 플로우 |

#### 핵심: 토큰을 DB에 저장하지 않는다

Executor는 **포인터(SecretRef)** 만 저장한다.

```
op://vault/github/token        ← 1Password
keychain://executor/slack      ← macOS 키체인
env://GITHUB_TOKEN             ← 환경변수
file://...                     ← 암호화 파일
```

호출하는 **그 순간에만** 신뢰 영역(프록시)에서 꺼내 헤더에 붙이고 버린다.
그래서 에이전트는 자격증명을 절대 볼 수 없다.

> 공식 문서 원문: *"Executor runs tool calls in a sandbox, so the agent can never
> access the credentials."*

시크릿 보관소 플러그인 5종: `onepassword`, `keychain`, `encrypted-secrets`,
`file-secrets`, `workos-vault`

### 레이어 2 — 내 에이전트 ↔ Executor (**상황에 따라 다름**)

| 실행 방식 | 토큰 필요? |
|---|---|
| 로컬 CLI / 데스크톱 | ❌ 불필요 (`127.0.0.1` 루프백) |
| Executor Cloud | ✅ 필요 (로그인 / 인증된 MCP 엔드포인트) |
| 자체 호스팅 | ✅ 필요 (외부 노출되므로) |

### 결론

- 로컬에서 공개 API만 쓸 거면 **토큰 없이 시작 가능**
- 내 계정 API를 붙이는 순간부터 그 API의 토큰 필요 (단, 안전하게 관리됨)
- **AI 모델 API 키(Claude/OpenAI)는 전혀 필요 없다.** Executor는 LLM을 호출하는
  주체가 아니라, LLM이 호출하는 도구다.

---

## 8. 왜 GitHub에서 유명한가

7개월 만에 ⭐3,912 (월평균 약 550개).

| # | 이유 | 설명 |
|---|---|---|
| 1 | **타이밍** | 2025~2026 MCP 대폭발기. "MCP 서버 10개 깔았는데 관리가 안 돼" 시점에 등장 |
| 2 | **N×M 문제 정조준** | 누구나 1초 만에 이해하는 고통을 해결 |
| 3 | **보안이 셀링포인트** | 에이전트가 키를 못 봄 + 도구별 정책 + 샌드박스 3종 |
| 4 | **코드 퀄리티** | Effect-TS 4.0 beta, Bun+Turborepo, e2e 완비, oxlint/oxfmt, 40+ 패키지 |
| 5 | **저자 인지도** | Rhys Sullivan (Answer Overflow 제작자) |
| 6 | **MIT + 선택지 5개** | 진입장벽 제로 |

> README 마지막에 저자가 직접 *"이 코드베이스를 레퍼런스로 써도 좋다"* 고 명시.
> 실제로 코드를 보려고 스타를 누르는 케이스가 많다.

---

## 9. 로컬 에이전트 구축에 도움이 되는가 → **매우 그렇다**

### 장점

| # | 내용 |
|---|---|
| 1 | **도구 연동 코드를 안 짜도 됨** — OpenAPI 파서, 인증 핸들러, 토큰 리프레시, 레이트리밋, JSON Schema 변환, 승인/재개 플로우 전부 내장 |
| 2 | **완전 로컬 실행** — 데이터가 밖으로 안 나감. 로컬 LLM + Executor = 오프라인 에이전트 스택 |
| 3 | **MCP 지원 에이전트면 즉시 연결** |
| 4 | **SDK 직접 임베드** — 서버 없이 인프로세스로 사용 가능 |
| 5 | **Human-in-the-loop 내장** — 위험 작업 자동 일시정지 → 승인 → 재개 |
| 6 | **Search & Invoke 모드** — 컨텍스트/토큰 비용 절감 |

```ts
import { createExecutor } from "@executor-js/sdk/promise";
import { openApiPlugin } from "@executor-js/plugin-openapi/promise";

const executor = await createExecutor({ plugins: [openApiPlugin()] });
const tools = await executor.tools.list({ integration: "inventory" });
const schema = await executor.tools.schema(tools[0].address);
await executor.close();
```

### Search & Invoke 모드 (`?mode=passthrough`)

도구가 수백 개면 프롬프트가 터진다. 이 모드는 도구를 **4개만** 노출한다.

| 도구 | 역할 |
|---|---|
| `integrations` | 연결된 계정 목록 (페이지네이션, 헬스 상태 포함) |
| `skills` | 이 서버의 가이드 읽기 |
| `search` | 의도로 도구 검색 + JSON 입력 스키마 조회 |
| `invoke` | 찾은 도구 ID로 실행 |

### 단점 / 주의사항

| 주의점 | 설명 |
|---|---|
| 의존성 추가 | Executor가 죽으면 도구 전부 멈춤 (SPOF) |
| 약간의 지연 | 프록시 한 단계 추가 |
| 학습 곡선 | Effect-TS 기반이라 코드 기여 진입장벽 있음 |
| **아직 베타** | `1.4.0-beta.0` — 프로덕션 도입은 신중히 |

---

## 10. React / PHP로 만들 수 있는가

질문을 두 가지로 나눠야 한다.

### A. "Executor 같은 걸 직접 만들 수 있나?"

| 언어 | 가능? | 평가 |
|---|:---:|---|
| TypeScript/React | ✅✅ | 이미 그렇다. 원본이 TS + React |
| PHP | ⚠️ | 가능하지만 비추천 |

**PHP 재구현의 난관**

- PHP는 요청-응답 모델 → MCP는 장수명 stateful 연결 필요
  (ReactPHP / Swoole / Workerman 등이 필요)
- JS 샌드박스 부재 — QuickJS / Deno / workerd 대체재 찾기 어려움
- MCP SDK 생태계가 TS·Python 대비 빈약
- 스트리밍(SSE), 백그라운드 데몬 구현이 까다로움

→ MIT로 이미 공짜인 것을 몇 달 걸려 재구현할 이유가 없다.

### B. "Executor를 React / PHP에서 쓸 수 있나?" → **이게 정답**

#### React — 최고 궁합

- `packages/react`에 이미 공유 React UI 컴포넌트가 있음
- `packages/app`이 React + TanStack Router
- React 19, Tailwind 4, Vite 8

할 수 있는 것: 커스텀 대시보드, 승인 UI(모바일 알림 형태), 채팅 에이전트 UI,
`packages/react` 컴포넌트 재활용.

#### PHP — 클라이언트로는 충분히 가능

```php
<?php
// PHP에서 Executor 도구 호출 (HTTP만 던지면 됨)
$ch = curl_init('http://127.0.0.1:4788/mcp');
curl_setopt($ch, CURLOPT_POST, true);
curl_setopt($ch, CURLOPT_HTTPHEADER, ['Content-Type: application/json']);
curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode([
  'jsonrpc' => '2.0',
  'id'      => 1,
  'method'  => 'tools/call',
  'params'  => [
    'name'      => 'github.issues.create',
    'arguments' => ['owner' => 'octocat', 'repo' => 'Hello-World', 'title' => 'Hi'],
  ],
]));
curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
$result = json_decode(curl_exec($ch), true);
```

라라벨/워드프레스 사이트에서 도구 호출, PHP 관리자 패널, 웹훅 → 워크플로 트리거 등.

#### 추천 조합

```
┌─────────────────────────────────────┐
│  Executor (그대로 사용, 수정 X)        │  ← 엔진
├─────────────────────────────────────┤
│  React 프론트엔드 (직접 제작)           │  ← 얼굴
│  + PHP/Laravel 백엔드 (선택)          │  ← 비즈니스 로직
└─────────────────────────────────────┘
```

---

## 11. 수익화 아이디어

### 법적 가능 여부: MIT라 완전 가능

| 허용 | 설명 |
|---|---|
| 상업적 판매 | 돈 받고 팔아도 됨 |
| 수정 | 자유롭게 |
| 비공개 배포 | 고친 걸 공개할 의무 없음 (GPL과 결정적 차이) |
| 리브랜딩 | 이름 바꿔 내 제품으로 |

**의무:** 저작권 표시 + 라이선스 사본 유지.
**주의:** "Executor" 이름/로고는 상표 이슈 가능 → 자체 브랜드명 권장.

### 이미 검증된 모델 (`autumn.config.ts` 실측)

| 플랜 | 가격 | 포함 |
|---|---|---|
| Free | $0 | 멤버 3명, 월 10만 실행 |
| Pay-as-you-go | 1,000실행당 $0.20 | 종량제 |
| Team | **시트당 월 $15** | 무제한 실행, 14일 체험 |
| Enterprise | 협의 | 무제한 + 도메인 인증 |

오픈소스 무료 + 클라우드 유료의 전형적인 오픈코어 모델.

---

### 아이디어 1: 한국 시장 특화 통합 팩 ⭐⭐⭐⭐⭐

원본 통합은 전부 미국 서비스(GitHub, Google, Microsoft, Stripe, Resend, WorkOS).
한국 기업이 실제로 쓰는 것은 다르다.

| 분야 | 한국 서비스 |
|---|---|
| 메신저 | 카카오톡 비즈메시지, 카카오워크, 네이버웍스 |
| 결제 | 토스페이먼츠, 포트원, KG이니시스, NHN KCP |
| ERP/회계 | 더존 iCUBE, 영림원, 이카운트, 삼쩜삼 |
| 물류 | CJ대한통운, 롯데택배, 스윗트래커 |
| 커머스 | 네이버 스마트스토어, 쿠팡 윙, 카페24, 고도몰 |
| 클라우드 | 네이버클라우드, 카카오클라우드, NHN클라우드 |
| 공공 | 공공데이터포털, 홈택스, 나라장터 |
| HR | 플렉스, 시프티, 뉴플로이 |

**수익 모델**

```
무료: 오픈소스 코어
유료: "K-통합팩" 구독
  ├─ 스타터      월 9만원   → 통합 5개
  ├─ 비즈니스    월 29만원  → 통합 20개 + 지원
  └─ 엔터프라이즈 협의       → 전체 + 온프레미스
```

**강력한 이유**

1. 경쟁자 없음 — 글로벌 업체는 시장 규모 때문에 한국 API를 안 만든다
2. 진입장벽이 곧 해자 — 한국 API는 문서가 부실하고 인증이 독특해 현지인만 가능
3. 가치 설명이 명확 — "우리 회사 ERP를 AI가 쓰게 해줌"
4. 기술적으로 쉬움 — OpenAPI 스펙만 만들면 Executor가 자동 도구화

> 핵심 인사이트: 한국 API 대부분은 OpenAPI 스펙이 없다.
> **"한국 API의 OpenAPI 스펙을 만드는 것" 자체가 상품이 된다.**

---

### 아이디어 2: 한국형 매니지드 호스팅 ⭐⭐⭐⭐

컨셉: "국내 리전 Executor Cloud" (네이버클라우드 / AWS 서울)

| 구매 이유 | 설명 |
|---|---|
| 규제 준수 | 개인정보보호법, 망분리, 전자금융감독규정 |
| 공공기관 | 국내 리전 필수, CSAP 인증 요구 |
| 레이턴시 | 미국 왕복 200ms → 국내 10ms |
| 한국어 지원 | 전화/카톡 문의 가능 |
| 세금계산서 | 해외 결제를 기피하는 기업이 많음 |

```
스타터       : 월 5만원      (5시트, 월 10만 실행)
팀           : 월 3만원/시트  (원본 $15 ≈ 2만원 + 국내 프리미엄)
엔터프라이즈 : 연 2,000만원~ (온프레미스 + SLA + 전담 지원)
```

"온프레미스 설치 + 연간 유지보수"는 한국 SI 시장에서 가장 잘 통하는 모델.

---

### 아이디어 3: 통합 팩 마켓플레이스 ⭐⭐⭐

Executor는 통합이 유일하게 열린 플러그인 이음새 → 통합 팩을 상품화할 수 있다.

| 패키지 | 내용 | 가격 |
|---|---|---|
| 커머스 팩 | 스마트스토어 + 쿠팡윙 + 택배3사 + 채널톡 | 29만원 |
| 병원 팩 | EMR + 예약 + 보험청구 + 카톡알림 | 99만원 |
| 스타트업 팩 | 슬랙 + 노션 + 리니어 + 깃헙 + 토스 | 19만원 |
| 법무 팩 | 등기소 + 대법원 + 계약관리 | 149만원 |
| 제조 팩 | 더존 + MES + 물류 + 세금계산서 | 199만원 |

수익 구조: 일회성 판매 + **구독(업데이트/API 변경 대응)** + 써드파티 수수료 30%.

> 한국 API는 자주 바뀐다. "계속 업데이트해드립니다"가 구독의 명분이자 지속 수익원.

---

### 아이디어 4: 컨설팅 / SI ⭐⭐⭐⭐

초기 자본 0원으로 지금 당장 현금이 되는 경로.

```
진단 워크샵          : 500만원 (2주)
PoC 구축            : 2,000만원 (1개월) — Executor + 통합 3개 + 에이전트 1개
본구축              : 5,000만원~ (3개월) — 전사 통합 + 권한체계 + 감사로그
연간 운영/유지보수   : 구축비의 15~20%/년  ← 알짜 수익
```

| 항목 | 직접 개발 | Executor 활용 |
|---|---|---|
| 개발 기간 | 6개월 | **1개월** |
| 인증/권한/감사 | 직접 구현 | 내장 |
| 샌드박스 보안 | 직접 구현 | 3종 내장 |
| 원가 | 높음 | 낮음 (마진 ↑) |

---

### 아이디어 5: 버티컬 AI 에이전트 제품 ⭐⭐⭐⭐⭐

Executor를 엔진으로 숨기고 완성품을 판다.

```
"통합 레이어 팝니다"      → 개발자만 이해함
"쇼핑몰 AI 매니저 팝니다"  → 사장님이 바로 이해함
```

**스마트스토어 AI 매니저** (월 9만 9천원, 타겟: 월매출 3천만원 이상 셀러)
- 주문 확인 → 재고 체크 → 송장 발행 자동화
- CS 문의 자동 답변 (배송조회 API 연동)
- 리뷰 분석 → 상품 개선 리포트
- 광고 성과 분석 → 예산 자동 조정

**병원 원무 AI** (월 49만원) — 예약 응대, 보험 청구, 노쇼 예측 + 카톡 리마인드

**중소기업 총무 AI** (월 29만원) — 세금계산서 정리, 급여/4대보험, 계약 만료 알림

| 항목 | 인프라 판매 | 버티컬 제품 |
|---|---|---|
| 고객 이해도 | 개발자만 | 누구나 |
| 지불 의사 | 낮음 | **높음** (인건비 대체) |
| 시장 크기 | 작음 | 큼 |
| 경쟁 | 글로벌 업체 | 로컬 우위 |

> "AI 직원 월 10만원" vs "알바 월 200만원" — 설득 논리가 명확하다.

---

### 아이디어 6: 틈새 기회

| 아이디어 | 설명 | 난이도 |
|---|---|---|
| 교육/강의 | "AI 에이전트 인프라 구축" 온라인 강의 | ⭐ |
| 템플릿/부트스트랩 | Executor + React 스타터킷 판매 | ⭐⭐ |
| 감사/모니터링 애드온 | 에이전트 행동 로그 분석 대시보드 (컴플라이언스) | ⭐⭐⭐ |
| 한국형 시크릿 볼트 | 국내 HSM/KMS 연동 플러그인 | ⭐⭐⭐⭐ |
| SI사 라이선싱 | 대형 SI에 화이트라벨 공급 | ⭐⭐⭐ |

---

### 단계별 로드맵

```
[1단계] 0~3개월 — 씨앗 뿌리기
├─ Executor 직접 써보며 숙달
├─ 한국 API 1~2개 통합 플러그인 오픈소스 공개 (인지도 + 포트폴리오)
└─ 블로그/유튜브 콘텐츠

[2단계] 3~6개월 — 첫 매출
├─ 컨설팅/PoC 수주 (자본 0원, 즉시 현금)
├─ 그 경험으로 "K-통합팩" 첫 버전 완성
└─ 고객 3곳 확보

[3단계] 6~12개월 — 제품화
├─ 버티컬 제품 1개 런칭 (커머스 또는 총무 추천)
├─ SaaS 구독 모델 전환
└─ 매니지드 호스팅 오픈
```

### 하나만 고른다면: **버티컬 제품 + 한국 통합팩**

- 지불 의사가 가장 높음 (인건비 대체 논리)
- 한국 API 진입장벽 = 자연 해자
- Executor가 90%를 해주므로 개발 부담이 적음
- SaaS 구독 = 반복 수익

### 리스크와 대응

| 리스크 | 대응 |
|---|---|
| 아직 베타 (`1.4.0-beta.0`) | 프로덕션 전 충분한 테스트, 버전 고정 |
| 원작자가 같은 시장 진입 | 한국 특화로 차별화 (글로벌 업체는 따라오기 어려움) |
| MIT 의무 | 저작권 표시 유지, 브랜드명 변경 |

---

## 12. 최종 요약

| 질문 | 답변 |
|---|---|
| 정체 | AI 에이전트용 오픈소스 통합 레이어. **MCP 서버 + MCP 클라이언트(프록시)** |
| 설치 | `npm i -g executor` → `executor install` → `executor web` (Node 20+) |
| 토큰 | 붙일 API의 토큰은 필요 / 로컬 Executor 자체는 불필요 / **AI 모델 키는 불필요** |
| 인기 이유 | ⭐3,912 (7개월). MCP 붐 + N×M 문제 + 보안 + 코드 퀄리티 + 저자 인지도 |
| 로컬 에이전트 | **매우 유용.** 도구/인증/승인/샌드박스 전부 제공. 단 아직 베타 |
| 수익화 | MIT라 가능. 한국 특화 SaaS / 호스팅 / 통합팩 / 컨설팅 / 버티컬 제품 |
| React/PHP | React = 최고 궁합 (이미 React 기반). PHP = 재구현 비추, 클라이언트로는 적합 |
