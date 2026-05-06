---
title: "Day 1: 환경 셋업 + 첫 에이전트"
nav_order: 3
---

# Day 1: 환경 셋업 + 첫 에이전트
{: .no_toc }

총 4시간 (강의·실습 3시간 + Q&A 1시간)
{: .fs-5 .fw-300 }

<details open markdown="block">
<summary>목차</summary>
{: .text-delta }
1. TOC
{:toc}
</details>

## 학습 목표

- AI 에이전트와 LLM의 차이를 본인의 언어로 설명한다
- OpenClaw를 본인 PC에 설치하고 게이트웨이를 띄운다
- 가장 단순한 에이전트를 1회 실행한다
- OpenClaw의 핵심 4개 개념(채널·세션·스킬·도구)을 이해한다

## 시간표

| 시간 | 블록 | 내용 |
|---|---|---|
| 0:00–0:50 | 1 | 오프닝 / 최종 데모 시연 / AI 에이전트 vs LLM / OpenClaw란? |
| 1:00–1:50 | 2 | OpenClaw 설치 / 첫 에이전트 1회 실행 |
| 2:00–2:50 | 3 | OpenClaw 핵심 개념 (채널·세션·스킬·도구) / clawhub 둘러보기 |
| 3:00–4:00 | Q&A | 셋업 보조 + 자유 질의응답 |

---

## Block 1: AI 에이전트와 OpenClaw 개념

### LLM과 AI 에이전트의 차이

#### 한 줄 비유

| LLM | AI 에이전트 |
|:---|:---|
| **자판기** 🥤 | **신입 인턴** 👩‍💼 |
| 동전(질문)을 넣으면 음료(답)가 나오는 1회 응답기 | 목표를 주면 도구를 알아서 찾아 쓰며 끝낼 때까지 일함 |

#### 실제 제품 예시

이미 들어봤을 도구로 비교해 보자.

| LLM (단순 응답) | AI 에이전트 (도구 사용 + 자율 행동) |
|:---|:---|
| **ChatGPT** — 질문하면 답하는 챗봇 | **Claude Code** — 터미널에서 코드를 짜고, 파일을 고치고, 명령어까지 직접 실행 |
| **Claude (claude.ai)** — 텍스트 대화 | **Cursor** — IDE에서 의도를 말하면 코드를 자동으로 수정 |
| **Gemini / 클로바X** — 채팅 | **Microsoft Copilot in Office** — 워드·엑셀 안에서 직접 문서를 만들고 편집 |
| **HuggingChat** — 모델만 호출하는 데모 | **OpenClaw + 메일 비서** *(이번 강의에서 만들 것)* — 메일을 직접 읽고 분류·이동·알림 |

> 💡 **같은 모델이라도 도구가 붙으면 에이전트가 된다.**
> Claude(LLM)와 Claude Code(에이전트)는 **두뇌가 같다**. 차이는 손발(도구)을 쥐어줬느냐다. ChatGPT와 ChatGPT의 GPTs/Deep Research 관계도 똑같다.
{: .note }

#### 작동 차이를 그림으로

**LLM 단독** — 입력→출력 1턴. 외부 세계에 닿을 손이 없음.

```mermaid
flowchart LR
    U([사용자]) -->|"오늘 받은 메일 분류해줘"| L[LLM]
    L -->|"메일에 접근할 수 없습니다"| U
    style L fill:#fbb,stroke:#a33,stroke-width:2px,color:#000
```

**AI 에이전트** — LLM이 두뇌가 되고, 도구로 실제 행동을 하며, 결과를 보고 다음 행동을 결정. 일이 끝날 때까지 반복.

```mermaid
flowchart LR
    U([사용자]) -->|"오늘 받은 메일 분류해줘"| A[AI 에이전트]
    A -->|"① 메일 가져와"| T1[메일 읽기 도구]
    T1 -->|"메일 5통"| A
    A -->|"② 이건 스팸인가?"| L[LLM 판단]
    L -->|"네, 스팸 2건"| A
    A -->|"③ 스팸 2건 이동"| T2[메일 이동 도구]
    T2 -->|"완료"| A
    A -->|"5통 처리 완료: 스팸 2 / 정상 3"| U
    style A fill:#bbf,stroke:#33a,stroke-width:2px,color:#000
    style L fill:#fbb,stroke:#a33,color:#000
    style T1 fill:#bfb,stroke:#3a3,color:#000
    style T2 fill:#bfb,stroke:#3a3,color:#000
```

#### 핵심 공식

> **AI 에이전트 = LLM(판단) + 도구(행동) + 루프(반복)**
{: .important }

이번 강의에서 OpenClaw로 만들 메일 비서가 정확히 이 구조다. LLM이 메일 내용을 보고 스팸인지 **판단**하고, 도구를 써서 실제로 메일을 **이동**시키고, 결과를 Discord로 **알린다**. 이 과정이 새 메일이 올 때마다 자동으로 **반복**된다.

### OpenClaw란?

자체 호스팅 가능한 **AI 비서 게이트웨이**. 한 줄로 요약하면:

> 평소 쓰는 메신저(Discord, Telegram 등)에서 AI 비서에게 일을 시키면, 그 비서가 **내 PC 안에서** 도구를 써서 일을 끝낸 뒤 같은 메신저로 결과를 보내주는 시스템.

#### 전체 구조

```mermaid
flowchart LR
    U([사용자]) -->|"Discord에서 명령"| Ch[채널: Discord]
    Ch --> Gate[OpenClaw 게이트웨이<br/>내 PC에서 동작]
    Gate --> Brain[LLM 두뇌<br/>Claude / GPT / Gemini]
    Gate --> Skills[스킬 + 도구]
    Skills -->|메일 IMAP| Ext1[(네이버 메일)]
    Skills -->|크롤링| Ext2[(웹사이트)]
    Skills -->|파일 I/O| Ext3[(로컬 파일)]
    Brain -.판단.-> Skills
    Skills -.결과.-> Brain
    Brain --> Gate
    Gate --> Ch
    Ch -->|"Discord 응답"| U
    style Gate fill:#bbf,stroke:#33a,stroke-width:3px,color:#000
    style Brain fill:#fbb,stroke:#a33,color:#000
    style Skills fill:#bfb,stroke:#3a3,color:#000
```

#### 무엇을 만들 수 있나?

OpenClaw로 다음 같은 비서를 만들 수 있다.

- 📧 **메일 비서** — 새 메일을 분류·요약하고 Discord로 알림 *(이번 강의에서 만들 것)*
- 📰 **뉴스 모니터** — 관심사 뉴스를 매일 아침 정리해서 보내줌
- ✍️ **콘텐츠 자동 발행** — 정리한 글을 네이버 블로그에 자동 업로드
- 📊 **데이터 모니터** — 공시·주가·재고 등이 변하면 즉시 알림
- 🧹 **파일 정리** — 다운로드 폴더를 종류별로 자동 분류·백업

#### 왜 "내 PC에서 돌리는" 게 의미 있나?

기존 ChatGPT는 **남의 클라우드**에서 동작한다. 메일을 분류하려면 메일 본문을 OpenAI 서버에 보내야 한다.

OpenClaw는 **내 PC 안에서 도는 LLM 비서**다.

- 🔒 **개인 데이터 안전** — 메일·캘린더·대화 내용이 외부로 나가지 않음
- 🛠 **자유로운 확장** — 내가 만든 도구·스킬을 마음대로 추가
- 🔁 **상시 가동** — 백그라운드 데몬으로 24시간 동작, 새 메일 즉시 반응

#### 핵심 4개 개념 (Block 3에서 자세히)

| 개념 | 한 줄 설명 |
|---|---|
| **채널** | 사용자가 에이전트와 대화하는 통로 (Discord, Slack, etc.) |
| **세션** | 한 번의 대화 맥락 (메모리 단위) |
| **스킬** | 자연어 runbook 형태의 작업 가이드 (`SKILL.md`) |
| **도구** | 코드로 작성된 호출 가능한 기능 (플러그인 또는 내장) |

---

## Block 2: 설치 + 첫 에이전트

> 이 섹션의 명령은 **Windows 10 기준**입니다. macOS/Linux 사용자는 일부 명령이 다릅니다.
{: .note }

### 2-1. Node.js + Git 설치 및 확인

OpenClaw는 **Node.js** 위에서 동작하고, 일부 명령에서 **Git**을 사용한다. [사전 과제](00-pre-assignment.html)에서 이미 설치하셨다면 [확인 부분](#확인)만 실행하면 됩니다.

#### 설치

PowerShell을 **관리자 권한**으로 열고:

```powershell
winget install -e --id OpenJS.NodeJS.LTS
winget install -e --id Git.Git
```

설치 후 PowerShell 창을 **닫고 새로 열어서** PATH 갱신을 반영합니다.

> winget이 없거나 동작하지 않으면 아래에서 installer를 직접 받아 설치하세요.
> - Node.js: [nodejs.org/ko/download](https://nodejs.org/ko/download) → "Windows Installer (.msi)" 64-bit
> - Git: [git-scm.com/download/win](https://git-scm.com/download/win)
{: .note }

#### 확인

새 PowerShell 창에서:

```powershell
node --version    # v20.x.x 이상
npm --version     # 10.x.x 이상
git --version     # git version 2.x.x
```

세 개 모두 버전이 떠야 다음 단계로 진행 가능합니다.

### 2-2. OpenClaw 설치

> ⏳ 이 섹션은 실제 VM 검증 후 정확한 명령으로 갱신됩니다. 현재 가설:
{: .warning }

```powershell
npm install -g openclaw
openclaw --version
```

### 2-3. 초기 설정 마법사

```powershell
openclaw doctor
```

`doctor`는 누락된 의존성·설정을 점검합니다. `--fix` 옵션을 추가하면 자동으로 고칠 수 있는 항목을 처리합니다.

### 2-4. 게이트웨이 시작

```powershell
openclaw gateway start
```

> ⏳ VM 검증 후 갱신: 게이트웨이 시작 후 어떤 화면이 보이는지, 종료 방법, 백그라운드 실행 옵션 등.
{: .warning }

### 2-5. 첫 에이전트 실행

> ⏳ VM 검증 후 갱신: 첫 명령(예: 인사 받기, 간단한 도구 사용) 시연.
{: .warning }

---

## Block 3: OpenClaw 핵심 개념

OpenClaw는 4개의 개념으로 모든 게 설명된다. **개인 비서를 둔 작은 사무실**에 비유해 보자.

| 개념 | OpenClaw 정의 | 일상 비유 |
|---|---|---|
| **채널** | 사용자와 에이전트의 입출력 통로 | 손님이 연락하는 **창구** (카톡·전화·이메일) |
| **세션** | 한 줄기 대화의 컨텍스트 단위 | 한 손님과의 **대화 한 줄기** (며칠짜리 카톡방처럼) |
| **스킬** | 자연어 작업 runbook (SKILL.md) | 직원이 따르는 **업무 매뉴얼** |
| **도구** | 코드로 호출 가능한 기능 | 직원이 손에 쥐는 **사무 비품** (프린터·전화·창고 키) |

전체 흐름:

```mermaid
flowchart LR
    User([사용자]) -->|메시지| Ch[채널<br/>Discord/Slack]
    Ch --> Sess[세션<br/>대화 맥락]
    Sess --> Agent[에이전트<br/>LLM 두뇌]
    Agent -->|"매뉴얼 참조"| Sk[스킬<br/>SKILL.md]
    Sk -->|"비품 호출"| Tool[도구<br/>코드 기능]
    Tool -->|실제 동작| External[(외부 시스템<br/>메일 / 파일 / 웹)]
    style Agent fill:#bbf,stroke:#33a,color:#000
    style Sk fill:#fec,stroke:#a83,color:#000
    style Tool fill:#bfb,stroke:#3a3,color:#000
```

### 채널 (Channel) — 창구

사용자가 에이전트와 대화하는 입출력 통로.

- **예**: Discord, Slack, Telegram, 터미널(CLI)
- 한 OpenClaw 게이트웨이에 **여러 채널을 동시에** 붙일 수 있음 (Discord와 Slack 둘 다 켜둬도 같은 비서가 응답)
- 채널마다 권한·필터를 따로 설정 (예: 특정 채널에서만 응답, 특정 사용자만 명령 가능)

> 비유: 사무실에 카톡·전화·이메일 창구가 모두 들어오지만 응대하는 비서는 한 명. 창구마다 "이 번호는 VIP 전용" 같은 룰을 따로 둘 수 있다.
{: .note }

### 세션 (Session) — 대화 한 줄기

한 줄기로 이어지는 대화의 컨텍스트 단위. 직전 대화를 기억하는 단위.

- DM(개인 메시지)·채널·사용자별로 세션이 분리됨
- 세션 안에서는 맥락 유지: **"방금 검색한 메일 요약해줘"** 같은 명령이 통함
- 채널마다 세션 정책을 다르게 줄 수 있음 (채널당 세션 1개 vs 사용자별 세션 분리)

> 비유: 친구 A와의 카톡방은 며칠 전 대화도 이어진다. 친구 B와의 카톡방과는 섞이지 않는다 — 각각 별도 세션이다.
{: .note }

### 스킬 (Skill) — 업무 매뉴얼

자연어로 작성된 작업 runbook. **코드 없이** 마크다운 파일로 에이전트의 행동을 정의한다.

- `SKILL.md` 파일 + (선택) 보조 스크립트
- YAML frontmatter로 이름·설명·필요한 환경변수, 마크다운 본문에 단계별 지시
- 에이전트가 사용자 메시지를 보고 적절한 스킬을 자동으로 호출

> 비유: "민원 접수 매뉴얼"처럼 직원이 따르는 업무 가이드. 새 매뉴얼 1개 = 새 스킬 1개. **이번 강의 Day 4에서 학생이 직접 SKILL.md를 작성한다.**
{: .important }

### 도구 (Tool) — 사무 비품

실제 코드로 구현된 호출 가능한 기능. 스킬은 도구를 써서 일을 끝낸다.

- **내장 도구**: `web_search`, `code_execution`, `cron`, 파일 I/O 등 OpenClaw 기본 제공
- **플러그인 도구**: TypeScript로 작성한 사용자 도구 (예: 네이버 메일 IMAP, 블로그 크롤러)
- **MCP 도구**: Model Context Protocol을 통해 연결된 외부 도구

> 비유: 매뉴얼이 "프린터로 출력해라" 하면 직원은 실제로 프린터(도구)를 작동시킨다. 매뉴얼 = 스킬, 프린터 = 도구.
{: .note }

### clawhub 살펴보기

`clawhub`은 OpenClaw 스킬·플러그인 카탈로그입니다.

> ⏳ VM 검증 후 갱신: `clawhub` 명령 동작 확인 + 사용 가능 스킬 리스트 캡처.
{: .warning }

---

## Day 1 산출물

- [ ] 모든 학생 PC에서 OpenClaw 게이트웨이 정상 시작
- [ ] 모든 학생이 에이전트와 1회 이상 대화 성공
- [ ] 4개 핵심 개념을 본인 언어로 설명 가능

다음: Day 2: 네이버 메일 연동 *(작성 예정)*
