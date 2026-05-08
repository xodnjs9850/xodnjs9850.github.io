---
title: 기본 설정
nav_order: 2
---

# 기본 설정
{: .no_toc }

Day 1 시작 전까지 아래 항목을 준비해주세요. 못 하신 분들도 Day 1 Q&A 시간(1시간)에 함께 마무리합니다.
{: .fs-5 .fw-300 }

<details open markdown="block">
<summary>목차</summary>
{: .text-delta }
1. TOC
{:toc}
</details>

## 체크리스트

- [ ] **(Windows)** WSL2 + Ubuntu 설치 + 첫 실행 완료
- [ ] **(macOS)** 터미널에서 `git --version`이 정상 출력 (없으면 Xcode CLT 설치)
- [ ] Gemini API 키 발급
- [ ] 네이버 메일 IMAP 활성화 + 앱 비밀번호 발급
- [ ] Discord 계정 + 본인 서버 1개 생성
- [ ] Discord 봇 생성 + 봇 토큰 확보 + 본인 서버에 봇 초대
- [ ] (선택) 메일 사고 방지용 `AI-Test` 폴더 생성

> 개인 메일 사고 방지를 위해, 강의 기간 중 자동 이동 대상 폴더는 네이버 기본 "스팸함"이 아닌 **별도 폴더(예: `AI-Test`)** 를 사용합니다. 본 페이지 끝에 폴더 만드는 방법이 있습니다.
{: .warning }

---

## 1. 환경 준비

OpenClaw는 **Linux 환경에서 가장 안정적**으로 동작합니다. Windows 사용자는 WSL2(Windows Subsystem for Linux) 위에 Ubuntu를 깔아 그 안에서 모든 명령을 실행합니다. macOS는 별도 처리 없이 native로 진행합니다.

### Windows: WSL2 + Ubuntu

#### Phase 1: WSL2 + 가상화 기능 활성화

PowerShell을 **관리자 권한**으로 열고 실행:

```powershell
wsl --install --no-distribution
```

이 명령이 가상 머신 플랫폼 + WSL2 커널을 활성화합니다. 끝나면 **재부팅 필요**.

#### Phase 2: Ubuntu 배포판 설치

재부팅 후 PowerShell **관리자 권한**으로 다시:

```powershell
wsl --install -d Ubuntu
```

Ubuntu 다운로드 + 설치 후 자동으로 Ubuntu 창이 열립니다 (안 열리면 시작 메뉴 → "Ubuntu" 검색).

#### Phase 3: Ubuntu 첫 실행

- **UNIX username**: 영문 소문자, 짧게 (예: `kw`)
- **UNIX password**: sudo 시 사용

bash 프롬프트(`kw@호스트:~$`)가 떠야 정상.

> WSL2 가상화 에러("WSL2 is not supported in this configuration" 또는 `HCS_E_HYPERV_NOT_INSTALLED`)가 나오면 BIOS에서 Intel VT-x / AMD-V가 켜져 있는지 확인하세요. 보통 최근 PC는 기본 켜져 있어 위 두 명령으로 해결됩니다.
{: .note }

> 호스트 Windows에 OpenClaw가 native로 설치돼 있다면, WSL2 안의 OpenClaw는 별도 환경(`~/.openclaw/` Ubuntu 내부)이라 충돌 없이 공존 가능합니다. 단, 동시에 Gateway를 켜면 같은 포트(`18789`)를 잡아 한쪽이 실패할 수 있어 한 번에 하나만 사용하세요.
{: .note }

### macOS

터미널에서 Git 설치 여부 확인:

```bash
git --version
```

안 떠 있으면 Xcode Command Line Tools 설치:

```bash
xcode-select --install
```

Node.js와 Homebrew 등 나머지 도구는 Day 1에서 함께 설치합니다.

---

## 2. Gemini API 키 발급

OpenClaw가 사용할 LLM(두뇌) 제공자입니다. Google AI Studio의 Gemini 무료 티어를 이용하면 강의 기간 내내 비용 0으로 진행 가능합니다.

1. [aistudio.google.com/apikey](https://aistudio.google.com/apikey) 접속
2. Google 계정 로그인
3. **Create API key** 클릭
4. (필요시) 새 Google Cloud 프로젝트 만들기 또는 기존 프로젝트 선택
5. 발급된 API 키를 안전한 곳에 보관 (예: 메모장, 비밀번호 관리자)

> Gemini 2.5 Flash는 일 1500 요청까지 무료 티어로 충분합니다. 강의용으로는 넉넉.
{: .note }

### 다른 LLM 제공자 (참고)

OpenClaw 2026.5.4은 약 40여 종의 LLM 제공자를 지원합니다. 강의는 **Gemini 무료 티어**를 가정하지만, 본인 환경/구독에 따라 다른 선택도 가능합니다.

| 카테고리 | 대표 제공자 | 비용 | 적합한 경우 |
|---|---|---|---|
| 🌟 **강의 권장** | Google (Gemini) | 무료 (1,500/day) | 가장 단순, 빠른 시작 |
| 💳 **보유 구독 활용** | Anthropic [Claude CLI](01-day1-setup.html#2-5-선택-pro-구독자용-claude-cli-경로), OpenAI Codex, GitHub Copilot | Pro·Plus 정액 | 이미 결제 중인 구독 재사용 |
| ⚡ **저렴/빠른 종량제** | DeepSeek, Groq, Cerebras, Together AI | 매우 저렴 / 무료 한도 큼 | 가성비·속도 |

전체 목록은 `openclaw setup --wizard`의 **Model/auth provider** 단계에서 검색 가능합니다 (검색창에 키워드 입력).

> 강의 자료는 Gemini를 가정해 작성되었지만, 다른 제공자도 OpenClaw 동작 자체는 동일합니다. 단, 무료 한도·응답 속도·모델 품질이 달라 데모용으로는 검증된 옵션(**Gemini 2.5 Flash**, Pro 구독자라면 **Claude Opus/Sonnet**)을 권장합니다.
{: .note }

---

## 3. 네이버 메일 IMAP + 앱 비밀번호

### 3-1. 2단계 인증 활성화

[네이버 메인](https://www.naver.com) → 우측 상단 본인 프로필 → **내 정보** → **보안 설정** → 2단계 인증 ON.

### 3-2. IMAP/SMTP 활성화

1. [네이버 메일](https://mail.naver.com) 접속
2. 왼쪽 하단 **환경설정** 톱니바퀴 → **POP3/IMAP 설정** 탭
3. **IMAP/SMTP 사용** → **사용함**
4. 저장

### 3-3. 애플리케이션 비밀번호 발급

1. [네이버 외부 앱 비밀번호 페이지](https://nid.naver.com/user2/help/myInfo.nhn?menu=pwdAlone) 접속
2. **애플리케이션 비밀번호 생성**
3. 용도 선택: "메일 (POP3/IMAP/SMTP)"
4. 발급된 16자리 비밀번호를 안전한 곳에 보관 (강의 중 사용)

> 이 비밀번호는 일반 네이버 비밀번호와 다르며, 2025년 6월부터 IMAP 접속에 필수입니다.
{: .important }

---

## 4. Discord 계정 + 서버

### 4-1. Discord 가입

[discord.com](https://discord.com)에서 계정을 만들고 데스크톱 앱 또는 브라우저로 로그인합니다.

### 4-2. 본인 서버 생성

1. 좌측 사이드바 맨 아래 **+ 버튼** → **나만의 서버 만들기**
2. 서버 이름: 자유 (예: `MyMailBot`)
3. 알림용 채널 1개 만들기 (예: `#mail-alerts`)

---

## 5. Discord 봇 생성 + 토큰 발급

### 5-1. 봇 애플리케이션 만들기

1. [Discord Developer Portal](https://discord.com/developers/applications) 접속
2. 우측 상단 **New Application** → 이름(예: `MailAssistant`) → Create
3. 좌측 **Bot** 메뉴 → **Reset Token** → 토큰 복사 (한 번만 보임, 안전 보관)

### 5-2. 권한(인텐트) 설정

같은 **Bot** 페이지에서 다음 인텐트를 켭니다:

- ✅ **PRESENCE INTENT**
- ✅ **SERVER MEMBERS INTENT**
- ✅ **MESSAGE CONTENT INTENT** (반드시!)

### 5-3. 봇을 본인 서버에 초대

1. 좌측 **OAuth2** → **URL Generator**
2. SCOPES: ✅ `bot`
3. BOT PERMISSIONS:
   - ✅ Read Messages/View Channels
   - ✅ Send Messages
   - ✅ Read Message History
4. 페이지 하단의 생성된 URL을 브라우저에서 열기 → 본인 서버 선택 → 인증

봇이 서버 멤버 목록에 (오프라인 상태로) 보이면 성공.

---

## 6. 개인 메일 사고 방지 — `AI-Test` 폴더 만들기

본 강의 기간 동안 OpenClaw가 분류하는 메일은 **이 폴더로** 이동시킵니다. 네이버 기본 스팸함은 일정 기간 후 자동 삭제되므로, 잘못 분류된 메일을 검토할 수 있도록 별도 폴더를 사용합니다.

1. [네이버 메일](https://mail.naver.com) 접속
2. 좌측 폴더 목록 하단 **+ 폴더 추가**
3. 이름: `AI-Test`
4. 만들기

---

## 자주 묻는 질문

**Q. Python은 안 깔아도 되나요?**
네. OpenClaw는 Node.js 기반이라 Python은 필요 없습니다.

**Q. Windows에서 왜 굳이 WSL2를 쓰나요?**
OpenClaw가 Linux 환경에서 가장 안정적으로 동작하고, Windows native에서는 Node.js의 `.cmd` 파일 spawn 이슈가 있어 일부 기능(Claude Code 연동 등)이 막힙니다. WSL2는 Win11에서 한 줄 명령으로 설치되며, macOS 사용자와 동일한 bash 터미널 경험을 제공해 강의 진행이 일관됩니다.

**Q. 봇 토큰을 잃어버렸어요.**
Developer Portal → Bot → **Reset Token**으로 새 토큰을 발급받으면 됩니다. 이전 토큰은 자동으로 무효화됩니다.

**Q. 앱 비밀번호와 일반 네이버 비밀번호 차이가 뭔가요?**
앱 비밀번호는 IMAP/SMTP 등 외부 앱 접속용 16자리 전용 비밀번호입니다. 일반 비밀번호는 IMAP 접속에 사용할 수 없습니다 (2025년 정책 변경).

**Q. Gemini 무료 티어 한도는 얼마나 되나요?**
모델·시점에 따라 다르지만 Gemini 2.5 Flash는 분당 15 요청, 일 1,500 요청까지 무료입니다 (변경 가능). 본 강의 분량으로는 넉넉합니다.
