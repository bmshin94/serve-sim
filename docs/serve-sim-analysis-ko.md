# serve-sim 분석 및 활용 정리 (한국어)

> 이 문서는 `serve-sim` 저장소를 처음 접한 사람을 위해 **이 프로젝트가 무엇인지, 어떻게 쓰는지,
> 어디에 도움이 되는지, 어떻게 확장·수익화할 수 있는지**를 정리한 자료입니다.

- **원본 저장소**: https://github.com/EvanBacon/serve-sim
- **이 저장소(포크)**: https://github.com/bmshin94/serve-sim
- **npm 패키지**: https://www.npmjs.com/package/serve-sim (`npx serve-sim`)
- **라이선스**: Apache-2.0
- **작성일**: 2026-09-18

---

## 1. 한 줄 요약

> **`npx serve`의 애플 시뮬레이터 버전.**
> 맥에서 돌아가는 iOS / iPadOS / watchOS 시뮬레이터를 **브라우저 URL 하나로 바꿔주는 도구**.

원본 제작자는 Expo 프레임워크 개발자인 Evan Bacon이며, 이 저장소는 그 포크본입니다.

---

## 2. 저장소 구조

| 경로 | 설명 |
|---|---|
| `packages/serve-sim/src/` | TypeScript 본체 (CLI · 서버 · 미들웨어), 약 31,000 LOC |
| `packages/serve-sim/src/client/` | React 19 + Tailwind 4 기반 웹 프리뷰 UI |
| `packages/serve-sim/src/__tests__/` | 80개 이상의 단위/E2E 테스트 |
| `packages/serve-sim/Sources/SimNative/` | Swift: 프레임 캡처, H.264 인코더, HID(터치) 주입, 접근성 브리지 |
| `packages/serve-sim/Sources/SimCameraInjector/` | ObjC dylib: AVFoundation 스위즐링으로 카메라 피드 대체 |
| `packages/serve-sim/Sources/SimCameraHelper/` | 호스트 측 카메라 헬퍼 (공유 메모리에 BGRA 프레임 기록) |
| `packages/serve-sim/Sources/SimAXSettings/` | 시뮬레이터 접근성 설정 조작 헬퍼 |
| `skills/serve-sim/` | Agent Skill (`SKILL.md` + references 6종 + 스크립트 + evals) |
| `.claude-plugin/` | Claude Code 플러그인 / 마켓플레이스 매니페스트 |
| `.github/workflows/` | lint · typecheck · unit · 실제 시뮬레이터 E2E · npm 배포 |

---

## 3. 동작 원리

```
┌──────────────┐   simctl io   ┌─────────────────┐  MJPEG / H.264 + WS  ┌─────────┐
│ iOS Simulator│ ────────────► │ serve-sim-bin   │ ───────────────────► │ Browser │
└──────────────┘    (Swift)    │ (디바이스당 1개) │                      └─────────┘
                               └─────────────────┘
                                       ▲
                                  상태 파일
                              $TMPDIR/serve-sim/
                                       ▲
                               ┌──────────────────┐
                               │ serve-sim CLI /  │
                               │ middleware       │
                               └──────────────────┘
```

1. **영상** — Swift 헬퍼가 `simctl io`로 프레임버퍼를 캡처 → H.264(또는 MJPEG) 인코딩 → HTTP 스트림.
2. **입력** — 브라우저의 클릭·키보드 → 바이너리 WebSocket → `HIDInjector`가 실제 터치 이벤트로 주입.
3. **무침습** — Xcode 플러그인도, 앱 코드 수정도 필요 없음. 부팅된 시뮬레이터면 모두 동작.

---

## 4. 주요 기능

- 60 FPS 스트리밍, 핀치 줌(Option 키), 바텀 스와이프 홈, 단축키 포워딩(⌘⇧H 등)
- **카메라 주입** — `DYLD_INSERT_LIBRARIES`로 AVFoundation 스위즐링, 이미지/영상/웹캠을 시뮬레이터 카메라로 공급
  (QR·바코드·OCR·얼굴인식 테스트에 유용)
- 드래그 앤 드롭으로 사진/영상을 시뮬레이터에 주입
- **접근성(AX) 트리 조회** — 픽셀 좌표 대신 시맨틱하게 요소 탐색
- 앱 권한(카메라/사진/위치/연락처/푸시) grant · revoke · reset
- CoreAnimation 디버그 오버레이, 메모리 경고 시뮬레이션, 화면 회전
- WebKit DevTools 프록시 (웹뷰 디버깅)
- 시뮬레이터 로그를 브라우저로 포워딩 → 에이전트가 읽어 판단
- iPhone / iPad / Apple Watch 지원, 다중 디바이스 동시 스트리밍

---

## 5. 설치 및 사용법

### 5.1 사전 요구사항

```sh
uname -m            # arm64 (Apple Silicon 전용, Intel Mac 미지원)
xcrun --version     # Xcode CLI 도구 (없으면 xcode-select --install)
node --version      # Node.js 20 이상 (유지보수 중인 LTS)
xcrun simctl list devices booted   # 부팅된 시뮬레이터 확인
```

카메라 주입 기능은 추가로 macOS 14 이상이 필요합니다.

### 5.2 실행

```sh
# 설치 없이 바로 실행 (권장)
npx serve-sim
# → Preview at http://localhost:3200

# 전역 설치
npm i -g serve-sim && serve-sim

# 이 저장소 소스에서 빌드 (macOS 필요)
bun install
bun run packages/serve-sim/build.ts
node packages/serve-sim/dist/serve-sim.js --port 3399
```

### 5.3 자주 쓰는 명령

```sh
serve-sim "iPhone 16 Pro"          # 특정 기기 지정
serve-sim --detach                 # 백그라운드 실행, JSON 반환 (에이전트용)
serve-sim --theme dark --fit       # 다크 모드 + 뷰포트 맞춤
serve-sim --list / --kill          # 실행 중 스트림 조회 / 종료

serve-sim tap 0.5 0.9              # 정규화 좌표(0..1) 단일 탭
serve-sim gesture '<json>'         # 드래그·멀티스텝 제스처
serve-sim type "hello"             # 텍스트 입력 (US 키보드)
serve-sim button home              # 하드웨어 버튼
serve-sim rotate landscape_left    # 화면 회전
serve-sim camera com.acme.App --file ~/qr.png   # 카메라 주입
serve-sim camera switch webcam     # 소스 핫스왑 (앱 재실행 없음)
serve-sim event-log                # 최근 시뮬레이터 이벤트
```

> 팁: 단순 탭은 `gesture`가 아니라 `tap`을 사용해야 합니다. `gesture`는 호출마다 별도 WebSocket을
> 열기 때문에 `begin`/`end`를 연속 호출하면 롱프레스로 인식될 수 있습니다.

### 5.4 기존 dev 서버에 임베드

```ts
import { simMiddleware } from "serve-sim/middleware";

app.use(simMiddleware({ basePath: "/.sim" }));
// → 프리뷰 HTML: /.sim,  상태 JSON: /.sim/api
```

원격 공유를 단일 포트로 하려면 `proxyHelpers: true`를 켜고 `upgrade` 이벤트를 배선해야 합니다.

```ts
const middleware = simMiddleware({ basePath: "/.sim", proxyHelpers: true });
app.use(middleware);
const server = app.listen(3000);
server.on("upgrade", (req, socket, head) => middleware.handleUpgrade(req, socket, head));
```

---

## 6. 플러그인 / 스킬 / MCP 구분

| 형태 | 제공 여부 | 설명 |
|---|:---:|---|
| npm CLI 패키지 | O (본체) | `serve-sim` — 실제 기능을 수행하는 주체 |
| Agent Skill | O | `skills/serve-sim/SKILL.md` — 에이전트에게 CLI 사용법을 가르치는 문서 |
| Claude Code 플러그인 | O | `.claude-plugin/plugin.json`, `marketplace.json` |
| MCP 서버 | **X** | MCP 프로토콜 구현체는 저장소에 존재하지 않음 |

```sh
# Claude Code
/plugin marketplace add EvanBacon/serve-sim
/plugin install serve-sim

# Agent Skills 표준을 지원하는 에이전트 (Cursor, Codex CLI, Gemini CLI 등)
bunx add-skill EvanBacon/serve-sim
```

MCP 대신 "CLI + SKILL.md" 조합을 택한 것은 의도적인 설계로 보입니다. MCP는 호스트별 설정이 갈리지만,
CLI는 Bash를 쓸 수 있는 모든 에이전트에서 동일하게 동작하기 때문입니다.
반대로 말하면 **MCP 래퍼는 아직 비어 있는 확장 지점**입니다.

---

## 7. API 토큰 / 보안

- **외부 API 토큰·계정·회원가입이 전혀 필요 없습니다.** 전 과정이 로컬에서 동작합니다.
- 내부적으로는 프로세스마다 **랜덤 세션 토큰**이 자동 생성되어 프리뷰 HTML에 주입되고,
  `/exec` 셸 실행 라우트와 WebSocket 제어 채널을 게이팅합니다
  (`packages/serve-sim/src/exec-ws.ts`, `middleware.ts`).
- 토큰 비교는 SHA-256 해싱 후 `crypto.timingSafeEqual`을 사용해 타이밍 공격을 방어합니다.

> **보안 주의**: 이 서버는 셸 명령을 실행할 수 있습니다. 코드 주석에도
> *"LAN — only on trusted networks"* 라고 명시되어 있습니다.
> 인터넷에 직접 노출하지 말고, 터널링할 경우 반드시 별도의 인증 계층
> (Cloudflare Access, ngrok basic auth 등)을 앞단에 두십시오.
> TLS를 리버스 프록시에서 종료할 때는 `X-Forwarded-Proto`를 전달해 mixed-content를 피해야 합니다.

---

## 8. 이 프로젝트가 주목받는 이유

1. Expo 핵심 개발자(Evan Bacon)가 만든 도구라는 신뢰도.
2. "시뮬레이터 화면을 공유·원격 조작하기 어렵다"는 iOS 개발자 공통의 실제 문제를 해결.
3. `npx` 한 줄로 끝나는 제로 설정 — 진입장벽이 사실상 없음.
4. AI 에이전트 붐과 "에이전트에게 GUI를 제공한다"는 흐름에 정확히 부합.
5. 카메라 주입처럼 기존에 불가능하던 기능을 구현 — 데모 임팩트가 큼.
6. `npx serve`에 빗댄 네이밍으로 한 문장 안에 가치 전달.
7. Swift 네이티브 + N-API + React 19 + Bun이라는 매력적인 기술 스택.
8. Apache-2.0 · 테스트 80개 이상 · CI 완비로 제품 수준의 완성도.

---

## 9. 로컬 에이전트 구축에 도움이 되는가

**도움이 되는 경우**

- macOS에서 iOS / React Native / Expo 앱을 개발하는 경우 — 사실상 필수 도구.
- AI가 앱을 실제로 조작하며 결과를 스스로 검증하는 루프를 만들고 싶은 경우.
- 시각적 회귀 테스트, QA 자동화를 에이전트에 위임하려는 경우.

**해당되지 않는 경우**

- 웹/백엔드 전용 개발 — Playwright 등이 더 적합.
- Android / Windows 대상 — `adb` 계열 도구 필요.
- macOS 장비가 없는 경우 — 서버 자체를 구동할 수 없음.

**부수적이지만 큰 가치: 아키텍처 레퍼런스**

| 배울 점 | 참고 위치 |
|---|---|
| 상태를 파일로 분리해 프로세스 간 느슨한 결합 구성 | `src/state.ts` |
| 데몬 모드 + JSON 출력(`--detach -q`)의 에이전트 친화적 CLI 설계 | `src/index.ts` |
| 로컬 서버 토큰 게이팅과 상수 시간 비교 | `src/exec-ws.ts` |
| WebSocket 멀티플렉싱으로 브라우저 6커넥션 제한 회피 | `src/exec-ws.ts` 상단 주석 |
| 에이전트용 문서(SKILL.md) 작성법 — When to use / When NOT / 안티패턴 | `skills/serve-sim/SKILL.md` |
| 접근성 트리 기반 요소 탐색 | `src/ax.ts`, `Sources/SimNative/AccessibilityBridge.swift` |

---

## 10. React / PHP로 만들 수 있는가

| 레이어 | 현재 기술 | React | PHP |
|---|---|:---:|:---:|
| 웹 UI | React 19 + Tailwind 4 | 이미 React | 가능하나 비권장 |
| CLI / 서버 | Node.js + TypeScript + `ws` | 가능 | 가능 (Ratchet, ReactPHP) |
| 화면 캡처 | Swift + CoreMedia | 불가 | 불가 |
| H.264 인코딩 | VideoToolbox | 불가 | 불가 |
| 터치 주입 | Swift HID (비공개 API) | 불가 | 불가 |
| 카메라 주입 | ObjC dylib + DYLD 스위즐링 | 불가 | 불가 |

핵심 레이어는 언어 선택의 문제가 아니라 **macOS 프레임워크 접근 권한**의 문제입니다.
따라서 다음과 같은 접근이 현실적입니다.

1. **UI 교체** — `src/client/` 전체가 React이므로 자체 브랜드 UI로 대체 가능. (가능)
2. **PHP 대시보드** — `serve-sim --detach -q`로 띄운 뒤 JSON/스트림을 PHP가 프록시. (가능)
3. **PHP/Laravel 오케스트레이터** — 여러 대의 맥에 있는 serve-sim 인스턴스를 관리·인증·과금. (가능, 사업성 있음)
4. 순수 JS/PHP로 캡처·입력 주입 재구현. (불가)

> 요약: **serve-sim은 엔진, 직접 만드는 것은 차체.**

---

## 11. 수익화 아이디어

Apache-2.0 라이선스이므로 상업적 이용·수정·비공개 재배포가 모두 허용됩니다.
LICENSE/NOTICE 유지, 변경 사실 고지, 공식 프로젝트 사칭 금지만 지키면 SaaS로 판매해도 문제없습니다.

**현재 비어 있는 지점(= 사업 기회)**
1. macOS 장비가 반드시 필요함
2. 원격 공유 시 인증·팀 협업 기능이 없음
3. MCP 서버, 다중 기기 오케스트레이션, 세션 녹화 기능이 없음

### 아이디어 1 — SimCloud: 클라우드 시뮬레이터 SaaS
- 맥 미니 여러 대에 serve-sim 인스턴스를 띄우고 웹에서 세션을 할당.
- PHP/Laravel이 적합한 영역: 인증, 세션 큐잉, 과금, 팀 관리.
- 가격 예시: 무료 30분/월 · Pro $19/mo · Team $99/mo · Enterprise 전용 맥.
- 타겟: Windows를 쓰는 디자이너·PM·QA, 부트캠프 수강생, 해외 외주팀.
- 리스크: 맥 하드웨어 초기 투자, Apple EULA(시뮬레이터는 macOS에서만 실행) 준수 필요.
- 난이도 상 / 수익 잠재력 매우 큼.

### 아이디어 2 — serve-sim MCP 서버 (우선 추천)
- serve-sim CLI를 MCP 툴로 래핑: `sim_tap`, `sim_screenshot`, `sim_camera`, `sim_ax_tree` 등.
- 오픈소스로 무료 배포해 인지도를 확보하고, Pro 기능(멀티 디바이스 매트릭스, 세션 녹화, 리포트)을 유료화.
- 가격 예시: 코어 무료 + Pro $9/mo, 또는 GitHub Sponsors.
- 타겟: Claude Code / Cursor / Codex를 쓰는 모바일 개발자 전반.
- 난이도 하 / 주말 2~3일 수준의 MVP / 인지도 확보 효과 큼.

### 아이디어 3 — 앱 데모 공유 서비스 (예: AppPreview.link)
- 앱 빌드(.app)를 업로드하면 시뮬레이터에 설치하고 스트림 공유 링크를 발급.
- 비밀번호·만료일 설정, 체험 세션 녹화 및 히트맵 분석 제공.
- 가격 예시: 링크 5개 무료 · Pro $29/mo · Agency $149/mo.
- 타겟: 인디 개발자, 외주사(클라이언트 리뷰), 스타트업 IR 데모.
- 킬러 유스케이스: 앱스토어 심사 없이 투자자·클라이언트에게 즉시 데모.
- 난이도 중 / 수익 잠재력 큼.

### 아이디어 4 — AI QA 자동화 플랫폼
- serve-sim + 접근성 트리 + LLM을 결합해 자연어 시나리오를 자동 실행.
- 예: "회원가입부터 결제까지 진행하고 막히는 지점을 알려줘" → 자동 탭·입력·스크린샷·리포트.
- 가격 예시: 월 100회 실행 $49/mo · 무제한 $299/mo.
- 타겟: 전담 QA가 없는 스타트업.
- 난이도 상 / 수익 잠재력 매우 큼 (인건비 대체이므로 지불 의사가 높음).

### 아이디어 5 — 카메라 테스트 특화 도구
- 카메라 주입 기능에 집중: QR/바코드 100여 종, 신분증 샘플, 얼굴 데이터셋, 저조도·흔들림 시나리오 라이브러리.
- CI에 연동해 카메라 회귀 테스트 자동화.
- 가격 예시: $39/mo 또는 에셋 팩 단건 판매($99~).
- 타겟: 핀테크(신분증 인증), 물류(바코드), 리테일(QR 결제).
- 난이도 하 / 경쟁자가 거의 없는 니치.

### 아이디어 6 — 교육 콘텐츠 및 컨설팅
- "AI 에이전트로 모바일 앱 테스트하기" 온라인 강의.
- 스킬/MCP 작성 워크숍, 기업 대상 도입 컨설팅.
- 한국어 문서·튜토리얼 선점을 통한 리드 확보.
- 난이도 하 / 즉시 착수 가능.

### 권장 로드맵

| 시점 | 실행 |
|---|---|
| 1개월 | 아이디어 2(MCP 서버) 오픈소스 공개 + 한국어 문서·블로그 → 비용 0, 인지도 확보 |
| 3개월 | 아이디어 5(카메라 테스트 팩) 유료화 → 첫 매출 |
| 6개월 | 아이디어 3 또는 4로 확장 → 웹 백엔드 역량 활용 구간 |
| 12개월 | 수요 검증 후 아이디어 1(클라우드)로 진입 |

핵심은 **serve-sim이 하지 않는 영역** — 인증, 팀 협업, 과금, 세션 관리, 리포팅 — 이며,
이는 모두 일반적인 웹 개발 역량으로 커버되는 부분입니다.

---

## 12. 참고 링크

- 원본 저장소: https://github.com/EvanBacon/serve-sim
- 이 저장소(포크): https://github.com/bmshin94/serve-sim
- npm: https://www.npmjs.com/package/serve-sim
- Agent Skills 문서: https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview
- 저장소 내 문서: `README.md`, `AGENTS.md`, `CLAUDE.md`, `skills/serve-sim/SKILL.md`
