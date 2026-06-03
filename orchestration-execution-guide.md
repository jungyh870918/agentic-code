# 에이전틱 오케스트레이션 — 실행 가이드

> 이 문서는 설계 논의를 종합한 실행 청사진입니다. 블로그/일지 작성은 컨텍스트에서 제외했습니다. 사람이 읽는 기록은 오직 **타 팀 교류용 이력**(PR 설명·이슈 코멘트·CHANGELOG·ADR) 목적으로만 남기며, 모두 GitHub 안에 인라인으로 둡니다.

---

## 0. 채팅방 구성 — 몇 개를 어떻게

**실행을 위해 새 채팅방을 만들지 않습니다.** 채팅방은 휘발성이라 진실의 원천이 될 수 없습니다. 에이전트의 실제 실행 무대는 GitHub Actions 안에서 호출되는 무상태 Claude API이고, 맥락은 매번 GitHub에서 읽어 주입합니다.

권장 구성은 두 개뿐입니다.

- **설계·상의 방** — 사람이 헤더와 구조를 정하고 막힌 문제를 푸는 자리 (지금 이 대화).
- **작업 지시 방** — 실제 구현 작업을 헤더에게 던지는 자리.

둘을 잇는 다리는 채팅 기억이 아니라 GitHub(이슈·PR·커밋)입니다. 셋 이상으로 늘릴 필요가 없습니다.

---

## 1. 에이전트 외주 분배

분배 기준은 **비용 × 신뢰 × 가역성**입니다. 비싼 프런티어 모델을 아무 데나 쓰면 비용이 새고, 싼 로컬 모델을 판단에 쓰면 품질이 무너집니다.

| 작업 성격 | 담당 | 이유 |
|---|---|---|
| 판단 무거움 (아키텍처·코드 리뷰·스펙 분해·영향 분석) | **Claude (헤더)** | 잘못된 판단의 복구 비용 > 호출 비용. 아끼지 않는다 |
| 생성 무겁고 검증 가능 (컴포넌트·보일러플레이트·UI) | **Cursor/v0 또는 Claude** | 테스트로 즉시 검증 가능 → 모델 위험 낮음 |
| 초안·발산 (기획 초안·카피·브레인스토밍·대안 탐색) | **ChatGPT / Gemini** | 어차피 헤더가 다듬음. 다양성이 장점 |
| 값싸고 반복적 (제목 정리·커밋 요약·라벨 분류·마스킹) | **M4 맥미니 로컬 LLM (Ollama)** | 정확도 영향 미미 + 호출 잦아 비용 절감 큼 |

**핵심:** 헤더(Claude)는 직접 일하지 않고 "이 작업을 누구에게 줄지"를 결정합니다. 이 라우팅 규칙 자체가 증명 포인트입니다.

**다만 Phase 1에서는 멀티 모델을 붙이지 않습니다.** 처음엔 Claude 하나로만 PR 리뷰를 돌리고, 멀티 모델 라우팅은 파이프라인이 안정된 Phase 2에서 도입합니다. 처음부터 네 모델을 엮으면 디버깅이 지옥이 됩니다.

---

## 2. 첫 단계 프로그램 설계 (Phase 1 — PR 자동 리뷰)

### 2.1 사전 준비물

1. GitHub 계정 + 새 **프라이빗 샌드박스 레포**
2. Anthropic API 키 (`console.anthropic.com`)
3. Node.js 환경 (맥미니에 설치)

> AWS는 Phase 1에서 **필요 없습니다.** 인프라는 Phase 3 전까지 미룹니다.

### 2.2 레포 골격 (최소)

```
repo-root/
├─ .github/workflows/pr-review.yml   PR 열릴 때 트리거되는 단 하나의 워크플로
├─ orchestrator/
│   ├─ review.mjs                     diff 읽고 Claude 호출, 코멘트 게시
│   └─ package.json                   @anthropic-ai/sdk 의존성
├─ CLAUDE.md                          에이전트 행동 규약 (권한·경계·가역성)
└─ src/                               리뷰 대상 데모 앱 코드 (점차 채움)
```

`apps/` `packages/` 등은 Phase 2에서 추가합니다.

### 2.3 세팅 순서

1. 위 골격을 빈 채로 레포에 올린다.
2. API 키를 `Settings → Secrets and variables → Actions`에 **`ANTHROPIC_API_KEY`** 로 등록한다. (코드에 절대 적지 않음)
3. `pr-review.yml`을 `pull_request` 이벤트에 반응하도록 설정한다.
4. 스크립트가 PR diff를 가져와 `CLAUDE.md`와 함께 Claude에 보내고, 리뷰를 PR 코멘트로 게시한다.

### 2.4 동작 흐름

```
PR 생성(사람) → Actions 기동(Secret 주입) → review.mjs(diff+규약 구성)
   → Claude API(리뷰 생성) → PR 코멘트 게시(기록 = 증명)
```

전 과정이 자유 가역(코멘트)이라 사람 게이트가 없습니다. 그러나 모든 단계가 GitHub에 남아 추적 가능합니다 — 어떤 diff를, 어떤 규약으로, 어떤 모델이 리뷰했는지.

---

## 3. 코드 골격

### 3.1 `.github/workflows/pr-review.yml`

```yaml
name: PR Auto Review
on:
  pull_request:
    types: [opened, synchronize]

permissions:
  contents: read
  pull-requests: write   # 코멘트 게시에 필요

jobs:
  review:
    runs-on: ubuntu-latest   # 나중에 맥미니 self-hosted 러너로 교체 가능
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0      # diff 비교를 위해 전체 히스토리

      - uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Install deps
        working-directory: orchestrator
        run: npm install

      - name: Run review
        working-directory: orchestrator
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}   # Actions가 자동 제공
          PR_NUMBER: ${{ github.event.pull_request.number }}
          REPO: ${{ github.repository }}
          BASE_SHA: ${{ github.event.pull_request.base.sha }}
          HEAD_SHA: ${{ github.event.pull_request.head.sha }}
        run: node review.mjs
```

### 3.2 `orchestrator/package.json`

```json
{
  "name": "orchestrator",
  "type": "module",
  "version": "0.1.0",
  "dependencies": {
    "@anthropic-ai/sdk": "^0.30.0"
  }
}
```

> 버전은 작업 시점에 `npm view @anthropic-ai/sdk version`으로 최신을 확인해 맞추세요.

### 3.3 `orchestrator/review.mjs`

```javascript
import Anthropic from "@anthropic-ai/sdk";
import { execSync } from "node:child_process";
import { readFileSync } from "node:fs";

const { ANTHROPIC_API_KEY, GITHUB_TOKEN, PR_NUMBER, REPO, BASE_SHA, HEAD_SHA } = process.env;

// 1) diff 추출 (자유 가역 — 읽기만)
const diff = execSync(`git diff ${BASE_SHA} ${HEAD_SHA}`, { maxBuffer: 10 * 1024 * 1024 }).toString();

// 2) 행동 규약 로드
let rules = "";
try { rules = readFileSync("../CLAUDE.md", "utf8"); } catch { rules = "(규약 파일 없음)"; }

// 3) Claude 호출 — 헤더의 판단
const client = new Anthropic({ apiKey: ANTHROPIC_API_KEY });
const msg = await client.messages.create({
  model: "claude-opus-4-7",          // 작업 시점 최신 모델 문자열로 갱신
  max_tokens: 2000,
  system: `너는 코드 리뷰 에이전트다. 아래 행동 규약을 따른다.\n\n${rules}\n\n` +
          `리뷰는 한국어로, 다음을 포함한다: (1) 변경 요약 (2) 발견한 문제와 근거 ` +
          `(3) 리스크 평가 (4) 머지 가능 여부 의견. 추측 금지, 근거만.`,
  messages: [{ role: "user", content: `다음 PR diff를 리뷰하라:\n\n\`\`\`diff\n${diff}\n\`\`\`` }],
});
const review = msg.content.filter(b => b.type === "text").map(b => b.text).join("\n");

// 4) PR 코멘트 게시 (관찰 가능성 — 누가·무엇을·왜를 기록)
const body = `## 자동 코드 리뷰 (Claude)\n\n${review}\n\n` +
             `---\n_모델: claude-opus-4-7 · 트리거: pull_request · 이 리뷰는 에이전트가 생성했습니다._`;

const res = await fetch(`https://api.github.com/repos/${REPO}/issues/${PR_NUMBER}/comments`, {
  method: "POST",
  headers: {
    "Authorization": `Bearer ${GITHUB_TOKEN}`,
    "Accept": "application/vnd.github+json",
    "Content-Type": "application/json",
  },
  body: JSON.stringify({ body }),
});

if (!res.ok) {
  console.error("코멘트 게시 실패:", res.status, await res.text());
  process.exit(1);
}
console.log("리뷰 코멘트 게시 완료.");
```

### 3.4 `CLAUDE.md` (행동 규약 — 추상 모델을 규칙으로)

```markdown
# 에이전트 행동 규약

## 권한 레이어 (누가 결정하는가)
- 의도 레이어: 사람 전용. 에이전트는 입력 못 함.
- 판단 레이어: 에이전트는 근거 제출까지, 결정은 사람. (머지·배포·아키텍처 승인)
- 실행 레이어: 에이전트 자율. (코드·리뷰·테스트·문서·IaC)
- 기계 레이어: 완전 자동. (빌드·린트·포맷)

## 신뢰 경계 (어디까지 내 영역인가)
- 내부 구역(내가 오너): 실행 자동 자유.
- 경계 구역(공유 코드·API 계약): 출력 전 사람 게이트.
- 외부 구역(타 팀·프로덕션): 읽기만, 쓰기·소통은 사람.

## 가역성 (게이트 라우팅)
- 자유 가역(코드·문서): 완전 자동.
- 비용 있는 가역(스테이징·테스트 데이터): 자동 + 로그.
- 준 비가역(프로덕션·운영 DB): 사람 게이트.
- 비가역(삭제·시크릿·결제): 게이트 + 추가 확인.

## 횡단 원칙
- 관찰 가능성: 모든 행동은 누가·무엇을·왜를 GitHub에 남긴다.
- 에스컬레이션: 막히면 멋대로 진행 말고 윗 레이어로 올린다. 테스트를 약화시켜 통과시키지 않는다.
- 상태/행동 분리: 진실의 원천은 GitHub 상태. 메모리에 들지 않는다.

## 타 팀 교류 이력 (사람이 읽는 기록)
- PR 설명에 변경 목적·영향 범위·근거를 사람이 읽을 수 있게 작성한다.
- 경계/외부 구역 변경은 사람이 타 팀에 소통한다. 에이전트는 초안만.
```

---

## 4. 검증 기준 (Phase 1 완료 조건)

- [ ] PR을 열면 1~2분 내 자동 리뷰 코멘트가 달린다.
- [ ] 코멘트에 모델·트리거·"에이전트 생성" 표기가 남는다 (관찰 가능성).
- [ ] API 키가 코드 어디에도 없고 Secret으로만 주입된다.
- [ ] 데모로 "이 리뷰는 사람이 쓰지 않았다"를 보여줄 수 있다.

이게 되면 다음 트리거(이슈→스펙 초안, 테스트 실패 분석)를 같은 패턴으로 추가합니다.

---

## 5. M4 맥미니 활용 (선택 — Phase 1 이후)

- **셀프 호스팅 러너**: `runs-on: ubuntu-latest`를 self-hosted로 교체 → CI 무제한·즉시 실행.
- **로컬 LLM(Ollama)**: 제목 정리·커밋 요약 등 값싼 보조 작업만. 핵심 판단은 Claude.
- **주의**: 맥미니를 회사 영구 인프라로 제안하지 않는다. 단일 장애점. "증명·개발 단계의 비용 절감 도구"까지가 적절한 위치. 처음부터 컨테이너로 짜서 클라우드 이전을 쉽게 한다.

---

## 6. 로드맵

| Phase | 기간 | 목표 |
|---|---|---|
| **1** (진행) | 2~3주 | PR 자동 리뷰, 이슈→스펙 초안, 테스트 실패 분석, Swagger 자동화. Claude 단일. |
| **2** | 3~4주 | 멀티 에이전트 라우팅 도입. 이슈→코드→리뷰→테스트→**머지 가능 상태**까지. 사람은 승인 버튼만. |
| **3** | 4~6주 | Terraform IaC, Secrets Manager, Prometheus+Grafana, 보안 스캔(ECR·SAST), 로드 테스트. |
