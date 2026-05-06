---
title: 사전 과제
nav_order: 2
---

# 사전 과제
{: .no_toc }

Day 1 시작 전까지 아래 4가지를 준비해주세요. 못 하신 분들도 Day 1 Q&A 시간(1시간)에 함께 마무리합니다.
{: .fs-5 .fw-300 }

<details open markdown="block">
<summary>목차</summary>
{: .text-delta }
1. TOC
{:toc}
</details>

## 체크리스트

- [ ] Node.js 20+ 설치
- [ ] 네이버 메일 IMAP 활성화 + 앱 비밀번호 발급
- [ ] Discord 계정 + 본인 서버 1개 생성
- [ ] Discord 봇 생성 + 봇 토큰 확보 + 본인 서버에 봇 초대
- [ ] (선택) 메일 사고 방지용 `AI-Test` 폴더 생성

> 개인 메일 사고 방지를 위해, 강의 기간 중 자동 이동 대상 폴더는 네이버 기본 "스팸함"이 아닌 **별도 폴더(예: `AI-Test`)** 를 사용합니다. 본 페이지 끝에 폴더 만드는 방법이 있습니다.
{: .warning }

---

## 1. Node.js 20+ 설치

OpenClaw 게이트웨이와 플러그인이 Node.js 위에서 동작합니다.

### Windows

PowerShell을 **관리자 권한**으로 열고 실행:

```powershell
winget install -e --id OpenJS.NodeJS.LTS
```

설치 후 PowerShell 창을 닫고 새로 열어 확인:

```powershell
node --version
npm --version
```

### macOS

```bash
brew install node
```

---

## 2. 네이버 메일 IMAP + 앱 비밀번호

### 2-1. 2단계 인증 활성화

[네이버 메인](https://www.naver.com) → 우측 상단 본인 프로필 → **내 정보** → **보안 설정** → 2단계 인증 ON.

### 2-2. IMAP/SMTP 활성화

1. [네이버 메일](https://mail.naver.com) 접속
2. 왼쪽 하단 **환경설정** 톱니바퀴 → **POP3/IMAP 설정** 탭
3. **IMAP/SMTP 사용** → **사용함**
4. 저장

### 2-3. 애플리케이션 비밀번호 발급

1. [네이버 외부 앱 비밀번호 페이지](https://nid.naver.com/user2/help/myInfo.nhn?menu=pwdAlone) 접속
2. **애플리케이션 비밀번호 생성**
3. 용도 선택: "메일 (POP3/IMAP/SMTP)"
4. 발급된 16자리 비밀번호를 안전한 곳에 보관 (강의 중 사용)

> 이 비밀번호는 일반 네이버 비밀번호와 다르며, 2025년 6월부터 IMAP 접속에 필수입니다.
{: .important }

---

## 3. Discord 계정 + 서버

### 3-1. Discord 가입

[discord.com](https://discord.com)에서 계정을 만들고 데스크톱 앱 또는 브라우저로 로그인합니다.

### 3-2. 본인 서버 생성

1. 좌측 사이드바 맨 아래 **+ 버튼** → **나만의 서버 만들기**
2. 서버 이름: 자유 (예: `MyMailBot`)
3. 알림용 채널 1개 만들기 (예: `#mail-alerts`)

---

## 4. Discord 봇 생성 + 토큰 발급

### 4-1. 봇 애플리케이션 만들기

1. [Discord Developer Portal](https://discord.com/developers/applications) 접속
2. 우측 상단 **New Application** → 이름(예: `MailAssistant`) → Create
3. 좌측 **Bot** 메뉴 → **Reset Token** → 토큰 복사 (한 번만 보임, 안전 보관)

### 4-2. 권한(인텐트) 설정

같은 **Bot** 페이지에서 다음 인텐트를 켭니다:

- ✅ **PRESENCE INTENT**
- ✅ **SERVER MEMBERS INTENT**
- ✅ **MESSAGE CONTENT INTENT** (반드시!)

### 4-3. 봇을 본인 서버에 초대

1. 좌측 **OAuth2** → **URL Generator**
2. SCOPES: ✅ `bot`
3. BOT PERMISSIONS:
   - ✅ Read Messages/View Channels
   - ✅ Send Messages
   - ✅ Read Message History
4. 페이지 하단의 생성된 URL을 브라우저에서 열기 → 본인 서버 선택 → 인증

봇이 서버 멤버 목록에 (오프라인 상태로) 보이면 성공.

---

## 5. 개인 메일 사고 방지 — `AI-Test` 폴더 만들기

본 강의 기간 동안 OpenClaw가 분류하는 메일은 **이 폴더로** 이동시킵니다. 네이버 기본 스팸함은 일정 기간 후 자동 삭제되므로, 잘못 분류된 메일을 검토할 수 있도록 별도 폴더를 사용합니다.

1. [네이버 메일](https://mail.naver.com) 접속
2. 좌측 폴더 목록 하단 **+ 폴더 추가**
3. 이름: `AI-Test`
4. 만들기

---

## 자주 묻는 질문

**Q. Python은 안 깔아도 되나요?**
네. OpenClaw는 Node.js 기반이라 Python은 필요 없습니다.

**Q. 봇 토큰을 잃어버렸어요.**
Developer Portal → Bot → **Reset Token**으로 새 토큰을 발급받으면 됩니다. 이전 토큰은 자동으로 무효화됩니다.

**Q. 앱 비밀번호와 일반 네이버 비밀번호 차이가 뭔가요?**
앱 비밀번호는 IMAP/SMTP 등 외부 앱 접속용 16자리 전용 비밀번호입니다. 일반 비밀번호는 IMAP 접속에 사용할 수 없습니다 (2025년 정책 변경).
