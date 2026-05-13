---
title: "Day 3: 스팸 분류 + 자동화 (cron)"
nav_order: 5
---

# Day 3: 스팸 분류 + 자동화 (cron)
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

- 스팸 분류 프롬프트를 설계해 LLM에 단일 메일 판단을 맡긴다
- 메일 이동 + 안전장치(`ai-test` 폴더, ham 우선)를 적용한다
- 분류·이동·로그를 한 번에 처리하는 **bash 스크립트**를 작성한다
- Ubuntu **crontab** 으로 주기적 자동 실행을 등록한다
- **자연어로 작성된 `SKILL.md`가 에이전트의 새 능력이 되는 패턴**을 직접 만들어본다

## 시간표

| 시간 | 블록 | 내용 |
|---|---|---|
| 0:00–0:50 | 1 | 스팸 분류 개념 + 안전장치 / 단일 메일 수동 분류·이동 |
| 1:00–1:50 | 2 | 분류 스크립트(`spam-filter.sh`) 작성 + 수동 실행 검증 |
| 2:00–2:50 | 3 | Ubuntu crontab 등록 / `linux-cron` SKILL.md 작성 / 자연어로 자동화 관리 |
| 3:00–4:00 | Q&A | 트러블슈팅 + 자유 질의응답 |

---

## Block 1: 스팸 분류 + 안전장치

### 메일 비서의 안전 원칙

이번 강의에서 만드는 자동 분류는 **본인 실제 메일**을 다룹니다. LLM이 판단하므로 100% 정확하지 않을 수 있어요. 이를 전제로 다음 원칙을 지킵니다.

| 원칙 | 이유 |
|---|---|
| **삭제하지 않고 이동** | 잘못 분류된 메일도 복구 가능 |
| **네이버 기본 스팸함이 아닌 `ai-test` 폴더로** | 네이버 스팸함은 일정 기간 후 자동 삭제 — `ai-test`는 사용자가 직접 검토 |
| **ham 우선 (모호하면 ham)** | False positive(정상 메일을 스팸으로)가 사용자 체감엔 더 나쁨 |
| **LLM은 판단만, 행동은 결정적 명령** | LLM hallucinate 차단 — 셸/도구가 실제 이동 실행 |

### 단일 메일 분류 실험

OpenClaw 메인 에이전트에 다음 프롬프트:

```
받은 편지함의 가장 최근 메일을 가져와서 스팸인지 ham인지 판단해줘.
JSON으로만 응답:
{"decision": "spam" or "ham", "reason": "한 줄 설명"}
```

응답 예시:

```json
{"decision": "spam", "reason": "발신자와 수신자가 동일하고, 무료 당첨금과 상품권을 미끼로 긴급한 클릭을 유도"}
```

LLM이 메일 내용을 보고 한 턴 안에 분류를 끝냅니다. 이 동작이 단위가 되어 자동화로 확장됩니다.

### 메일 이동 (수동)

himalaya CLI로 직접:

```bash
# 받은 편지함에서 ai-test로 이동
himalaya message move ai-test <ID>

# 결과 확인
himalaya envelope list -f ai-test
```

또는 에이전트에 자연어로:

```
받은 편지함 가장 최근 메일을 스팸 분류하고, spam이면 ai-test 폴더로 이동해줘.
```

→ 에이전트가 [Day 2](02-day2-mail.html)에서 설치한 himalaya 스킬을 사용해 분류·이동을 한 번에 수행.

---

## Block 2: 분류 스크립트 작성 + 수동 검증

Block 1의 한 통 분류를 **여러 통씩, 결정적으로** 처리하도록 bash 스크립트로 옮깁니다. LLM은 분류만, 셸이 이동·로깅을 직접 수행하므로 LLM의 환각이나 누락에 영향을 받지 않습니다.

### 왜 OpenClaw cron이 아니라 Ubuntu crontab인가

OpenClaw 자체 cron(`openclaw cron add`)도 있지만, 강의 경험상 다음 이유로 **OS의 표준 cron** 을 권장합니다:

| | OpenClaw cron | Ubuntu crontab (권장) |
|---|---|---|
| 인증·세션 | 격리 세션이 매번 새로 인증을 시도 (실패 잦음) | 사용자 셸 환경 그대로 상속 (안정적) |
| 디버깅 | 격리 세션 추상화 안에서 일어남 | 스크립트를 수동 실행하면 100% 동일 재현 |
| LLM provider 한도/billing 영향 | cron이 직접 LLM 호출 → 한도/결제 이슈 즉시 노출 | 스크립트가 OpenClaw를 한 단계 거쳐 호출 → 라우팅 위임 |
| 학생 평생 자산 | OpenClaw 종료 시 사용 불가 | Linux 시스템 표준, 어디서나 쓸 수 있는 지식 |

OpenClaw의 역할은 **LLM 라우터 + 채널 통합 + 스킬 정의**에 집중하고, "언제 돌릴지" 는 OS 자체 기능에 위임하는 분업이 깔끔합니다.

### 2-1. 스크립트 폴더 + Discord webhook 자리(선택) 준비

```bash
mkdir -p ~/.openclaw/scripts ~/.openclaw/logs ~/.openclaw/secrets
```

Discord 알림은 Day 4에서 본격 다룹니다. 이번 Day는 **터미널 출력**만으로 진행하되, 스크립트는 webhook URL이 환경변수에 있으면 자동으로 Discord로 보내도록 만들어둡니다.

```bash
# Day 4에서 채울 자리만 만들기 (지금은 비워둠)
touch ~/.openclaw/secrets/discord.env
chmod 600 ~/.openclaw/secrets/discord.env
```

이미 [Day 1](01-day1-setup.html) 등에서 `~/.bashrc` 에 secrets source 라인이 들어가 있다면 그대로 사용. 아직 없다면:

```bash
cat >> ~/.bashrc <<'EOF'

# OpenClaw secrets
[ -f ~/.openclaw/secrets/discord.env ] && . ~/.openclaw/secrets/discord.env
EOF
source ~/.bashrc
```

### 2-2. `spam-filter.sh` 작성

`~/.openclaw/scripts/spam-filter.sh` 파일에 아래 내용을 넣습니다. `nano` 로 열어 그대로 붙여넣기 + 저장(`Ctrl+O`, Enter, `Ctrl+X`):

```bash
nano ~/.openclaw/scripts/spam-filter.sh
```

> 네이버 메일은 자체 **스마트메일함**이 도착 메일을 `프로모션`, `뉴스레터함`, `쇼핑레터함` 등으로 자동 분배합니다. 그래서 `INBOX` 만 보면 광고가 거의 안 잡혀요. 강의 baseline은 INBOX와 광고성 폴더 3개를 함께 검사합니다. 본인 환경에 폴더가 다르면 `SOURCE_FOLDERS` 배열을 수정하세요 (`himalaya folder list -a <ACCOUNT>` 로 확인).
{: .note }

```bash
#!/usr/bin/env bash
# spam-filter.sh — 여러 폴더의 최근 N통을 분류, spam은 ai-test로 이동, 결과 출력/알림
# LLM 라우팅은 OpenClaw 책임 (모델 명시 X), Gateway 경유
set -euo pipefail

# === 설정 ===
ACCOUNT="naver"                 # 본인 himalaya 계정 이름과 일치시키기 (Day 2에서 정한 이름)
# 검사할 폴더 목록 — 네이버 스마트메일함이 이미 광고를 분류해두는 환경이라 INBOX 외에 광고성 폴더도 함께 본다.
# 본인 환경에 맞춰 추가/삭제 가능. 폴더 이름은 `himalaya folder list -a <ACCOUNT>` 로 확인.
SOURCE_FOLDERS=("INBOX" "프로모션" "뉴스레터함" "쇼핑레터함")
TARGET_FOLDER="ai-test"
BATCH_SIZE=5                    # 폴더당 N통 (폴더 수 × BATCH_SIZE 만큼 LLM 호출 — 너무 크면 한 사이클이 길어짐)
LOG_DIR="$HOME/.openclaw/logs"
LOG_FILE="$LOG_DIR/spam-filter.jsonl"

mkdir -p "$LOG_DIR"
ts() { date -Iseconds; }

checked=0; spam_count=0; ham_count=0; moved=0

# === 폴더별 반복 ===
for SOURCE_FOLDER in "${SOURCE_FOLDERS[@]}"; do
  envelopes=$(himalaya --output json envelope list -a "$ACCOUNT" -f "$SOURCE_FOLDER" -s "$BATCH_SIZE" 2>/dev/null || echo "[]")
  mapfile -t ids < <(echo "$envelopes" | jq -r '.[].id')

  for id in "${ids[@]}"; do
    meta=$(echo "$envelopes" | jq -c ".[] | select(.id == \"$id\")")
    subject=$(echo "$meta" | jq -r '.subject')
    from=$(echo "$meta" | jq -r '.from.addr // .from.name // "unknown"')
    body=$(himalaya message read "$id" -a "$ACCOUNT" -f "$SOURCE_FOLDER" 2>/dev/null | head -c 2000 || echo "")

    prompt="당신은 한국어 메일 분류기입니다. 다음 메일을 'spam' 또는 'ham'으로 분류하세요.

[즉시 spam — 다른 기준 무시]
1. 제목이 (광고), [광고], (Ad), [Ad], (Sponsored) 등으로 시작 (한국 정보통신망법상 광고 메일은 제목에 (광고) 표시가 의무)
2. 제목에 다음 마케팅 키워드 중 하나라도:
   할인, 세일, 쿠폰, 이벤트, 당첨, %할인, 최대 N%, 마지막 기회, 오늘만, 한정, 특가, 무료
3. 본문이 거의 비어있거나 짧은 HTML 조각만 있는 경우 (이미지로만 구성된 광고 메일의 전형적 패턴)
→ 위 셋 중 하나라도 해당하면 본문 내용·발신자 도메인과 무관하게 spam

[분류 기준]
- ham: 본인이 한 행동에 대한 응답 메일(영수증·결제·인증·배송·계정 보안·약관 변경 등), 지인/업무 메일, 본인이 직접 요청한 응답
- spam: 광고·마케팅·프로모션·할인·세일·이벤트 안내·뉴스레터, 당첨 미끼, 피싱, 금융사칭, 위장 발신자
- 핵심: 본인이 가입한 정상 도메인의 메일이라도 마케팅·구매 유도성이면 spam
- 위 기준으로도 모호하면 ham

[안전 원칙]
- 오늘 날짜는 $(date +%Y-%m-%d) 입니다. 메일 날짜가 학습 데이터 기준 미래여도 환각이 아니라 실제 시점입니다.
- 본문이 짧거나 비어있어도 제목·발신자만으로 판단 가능합니다.

[출력 형식]
JSON 한 줄만 출력. 마크다운 코드 펜스로 감싸지 마세요. 다른 설명 금지.
{\"decision\":\"spam\"|\"ham\",\"reason\":\"한국어 한 줄\"}

[입력 메일]
제목: $subject
발신: $from
본문(앞부분): $body"

    # 모델 명시 X — OpenClaw 라우팅에 위임
    raw=$(openclaw infer model run --gateway --json --prompt "$prompt" 2>/dev/null || echo '{}')

    # OpenClaw 응답 → .outputs[0].text 안에 LLM JSON 문자열 → 두 번 파싱
    decision=$(echo "$raw" | jq -r '.outputs[0].text | fromjson? | .decision // "ham"')
    [ -z "$decision" ] && decision="ham"

    checked=$((checked + 1))
    if [ "$decision" = "spam" ]; then
      himalaya message move "$TARGET_FOLDER" "$id" -a "$ACCOUNT" -f "$SOURCE_FOLDER" >/dev/null 2>&1 && moved=$((moved + 1))
      spam_count=$((spam_count + 1))
    else
      ham_count=$((ham_count + 1))
    fi

    printf '{"ts":"%s","folder":"%s","id":"%s","subject":%s,"decision":"%s"}\n' \
      "$(ts)" "$SOURCE_FOLDER" "$id" "$(jq -Rs . <<<"$subject")" "$decision" >> "$LOG_FILE"
  done
done

# === 3. 결과 알림 ===
msg=$(printf '**[spam-filter]** %s\n검사 %d통 / 스팸 %d / ham %d\n이동 완료: %d통 → `%s`' \
  "$(date '+%Y-%m-%d %H:%M')" "$checked" "$spam_count" "$ham_count" "$moved" "$TARGET_FOLDER")

# webhook URL이 있으면 Discord로, 없으면 터미널에 출력
if [ -n "${DISCORD_WEBHOOK_URL:-}" ]; then
  payload=$(jq -nc --arg c "$msg" '{content: $c}')
  curl -sS -X POST -H "Content-Type: application/json" -d "$payload" "$DISCORD_WEBHOOK_URL" >/dev/null
else
  echo "$msg"
fi
```

> 스크립트 안의 `ACCOUNT="naver"` 는 본인이 Day 2 himalaya 마법사에서 정한 **계정 이름**과 정확히 일치해야 합니다. 다르면 `himalaya account list` 로 확인 후 수정.
{: .warning }

권한 부여:

```bash
chmod +x ~/.openclaw/scripts/spam-filter.sh
```

### 2-3. CRLF 함정 회피

Windows 측에서 스크립트를 편집해 WSL로 가져온 경우 줄바꿈이 `\r\n` 으로 저장돼 `env: 'bash\r': No such file or directory` 같은 에러가 납니다. **WSL Ubuntu 안에서 `nano` 로 직접 작성**하면 이 문제가 없습니다. 만일 발생하면:

```bash
sed -i 's/\r$//' ~/.openclaw/scripts/spam-filter.sh
```

### 2-4. 수동 실행

OpenClaw Gateway가 떠 있는 상태에서 (Day 1에서 띄워둔 상태 그대로):

```bash
~/.openclaw/scripts/spam-filter.sh
```

한 통당 5~15초 정도 LLM 호출이 일어나므로 10통이면 1~2분 소요. 끝나면 터미널에:

```
**[spam-filter]** 2026-05-13 10:38
검사 10통 / 스팸 1 / ham 9
이동 완료: 1통 → ai-test
```

같은 줄이 출력됩니다. 로그 파일에는 한 통씩 결정 결과가 JSONL 형태로 누적:

```bash
tail -n 10 ~/.openclaw/logs/spam-filter.jsonl
```

```json
{"ts":"2026-05-13T10:38:14+09:00","id":"23279","subject":"\"디즈니+ 약관 및 정책 변경 안내\"","decision":"ham"}
{"ts":"2026-05-13T10:38:25+09:00","id":"23277","subject":"\"...휴면회원 개인정보 파기 예정 안내\"","decision":"ham"}
...
```

### 2-5. 메일 상태 확인

```bash
himalaya envelope list -a naver -s 5            # INBOX
himalaya envelope list -a naver -f ai-test -s 5 # 이동된 메일
```

spam으로 판정된 메일이 ai-test 폴더에 도착했으면 성공.

---

## Block 3: Ubuntu crontab + `linux-cron` SKILL.md

### 3-1. crontab 등록 — 1시간마다 자동 실행

```bash
crontab -e
```

기본 에디터(nano)가 열리면 맨 아래에 한 줄 추가:

```
0 * * * * BASH_ENV=$HOME/.openclaw/secrets/discord.env bash -lc '$HOME/.openclaw/scripts/spam-filter.sh' >> $HOME/.openclaw/logs/spam-filter.cron.log 2>&1
```

- `0 * * * *` — 분=0, 시·일·월·요일 모두(`*`) 즉 **매 정시**
- `BASH_ENV=...` — cron 은 비대화형 셸이라 `~/.bashrc` 가 자동 source 안 됨. 시크릿 파일을 직접 지정해서 환경변수 주입
- `bash -lc` — 로그인 셸로 실행해 `PATH`(himalaya, openclaw, jq) 정상 로드
- `>> ... 2>&1` — 표준 출력/에러를 cron 로그 파일로

저장 후 등록 확인:

```bash
crontab -l
```

위 한 줄이 그대로 보이면 완료. **다음 정시**(예: 11:00, 12:00 …)에 자동 실행됩니다.

#### 다른 주기 표기

| 사람말 | crontab |
|---|---|
| 매 정시 (1시간마다) | `0 * * * *` |
| 매 30분 | `*/30 * * * *` |
| 매 15분 | `*/15 * * * *` |
| 매 5분 | `*/5 * * * *` |
| 매일 오전 9시 | `0 9 * * *` |
| 평일 오전 9시만 | `0 9 * * 1-5` |

### 3-2. 자동 사이클 검증

다음 정시까지 기다린 후:

```bash
tail -n 20 ~/.openclaw/logs/spam-filter.jsonl
```

새 분류 결과가 누적됐으면 cron이 정상 발화한 것. (스크립트가 침묵 운영이라 `spam-filter.cron.log` 가 비어 있을 수 있는데, 이는 정상입니다 — 결과는 JSONL 쪽에 들어가요.)

cron daemon 자체가 발화했는지 보고 싶다면:

```bash
journalctl -u cron --since "today 00:00" | grep spam-filter | tail -5
```

`(<사용자>) CMD (BASH_ENV=... bash -lc '...')` 같은 줄이 보이면 OS 레벨에서 정확히 트리거된 것.

### 3-3. `linux-cron` SKILL.md 작성 — Day 3의 백미

여기서부터가 **OpenClaw 생태계의 핵심 가치**를 체험하는 부분입니다. 위에서 만든 crontab 운영을 봇이나 메인 에이전트가 **자연어로 처리**할 수 있도록 매뉴얼 한 장을 작성합니다.

```bash
mkdir -p ~/.openclaw/workspace/skills/linux-cron
nano ~/.openclaw/workspace/skills/linux-cron/SKILL.md
```

내용:

```
---
name: linux-cron
description: Ubuntu crontab으로 메일 분류 cron(spam-filter) 운영. 스케줄 확인·주기 변경·수동 실행·로그 조회·삭제.
---

# linux-cron — Ubuntu crontab 운영 비서

사용자의 메일 분류 자동화는 Ubuntu의 표준 crontab 으로 돌아갑니다.

- 스크립트: $HOME/.openclaw/scripts/spam-filter.sh
- 분류 로그 (JSONL): $HOME/.openclaw/logs/spam-filter.jsonl
- cron stdout 로그: $HOME/.openclaw/logs/spam-filter.cron.log

## 언제 이 스킬을 사용하나

다음 의도가 보이면:

- 스케줄 조회 — "스케줄 확인", "cron 보여줘", "예약 알려줘"
- 주기 변경 — "주기 1시간으로 바꿔줘", "더 자주 돌게", "30분마다"
- 수동 실행 — "지금 돌려봐", "한번 실행해봐", "수동 실행"
- 로그 조회 — "최근 분류 결과", "어떤 메일 옮겼어", "오늘 통계"
- 삭제 — "자동 분류 멈춰", "cron 삭제"
- 재등록 — "다시 등록해줘", "1시간마다 돌게 만들어"

## 도구 매핑

| 의도 | 명령 |
|---|---|
| 조회 | crontab -l |
| 수동 실행 | bash $HOME/.openclaw/scripts/spam-filter.sh & |
| 최근 분류 N건 | tail -n 20 $HOME/.openclaw/logs/spam-filter.jsonl |
| 오늘 통계 | grep "$(date +%Y-%m-%d)" $HOME/.openclaw/logs/spam-filter.jsonl | jq -r .decision | sort | uniq -c |
| 삭제 | crontab -l | grep -v spam-filter | crontab - |
| 등록 (1시간 주기 기본) | (crontab -l 2>/dev/null | grep -v spam-filter; echo "0 * * * * BASH_ENV=$HOME/.openclaw/secrets/discord.env bash -lc '$HOME/.openclaw/scripts/spam-filter.sh' >> $HOME/.openclaw/logs/spam-filter.cron.log 2>&1") | crontab - |
| 주기 변경 | 등록 명령에서 5필드 부분만 새 주기로 바꿔 재실행 |
| daemon 실행 추적 | journalctl -u cron --since "today 00:00" | grep spam-filter | tail -10 |

## 응답 형식

표 + 친근한 한 줄 해설.

## 규칙

1. 변경 전 반드시 현재 상태를 보여주고 사용자 확인을 받는다.
2. 수동 실행은 `&` 로 백그라운드 + "잠시 후 결과가 도착할 거예요" 안내.
3. spam-filter 단어로 필터링한 줄만 수정/삭제. 다른 cron 줄은 건드리지 않는다.
4. 시스템 cron(/etc/cron.*)은 만지지 않는다. 사용자 crontab 안에서만.
5. cron.log 가 비어있어도 정상 — 실제 결과는 spam-filter.jsonl 에 누적.
6. JSONL 로그를 사람 읽기 쉬운 표로 변환.
```

> SKILL.md를 다른 환경(예: Windows 메모장)에서 작성한 뒤 WSL로 가져왔다면 CRLF가 섞일 수 있어요. `sed -i 's/\r$//' ~/.openclaw/workspace/skills/linux-cron/SKILL.md` 로 정리.
{: .warning }

### 3-4. SKILL 인식 + 검증

Gateway가 시작할 때 SKILL을 로드하므로 재시작:

```bash
# 백그라운드 모드
systemctl --user restart openclaw-gateway

# 또는 foreground 모드면 해당 터미널에서 Ctrl+C 후 다시:
openclaw gateway
```

스킬 상태 확인:

```bash
openclaw skills info linux-cron
```

`Visible to model: yes` 가 보여야 정상.

OpenClaw 메인 에이전트에서 자연어로:

```
스케줄 확인해줘
```

응답 예시:

```
현재 스케줄

| 작업         | 주기      | 다음 실행 | 최근 실행 | 상태 |
|--------------|-----------|----------|----------|------|
| spam-filter  | 매 정시   | 47분 후   | 13분 전   | OK   |

매 정시마다 INBOX·프로모션·뉴스레터함·쇼핑레터함의 최근 5통씩(총 20통)을 검사 → spam은 ai-test 폴더로 이동합니다.
```

추가 시나리오:

```
오늘 분류 결과 통계 알려줘
최근 분류 로그 보여줘
지금 한번 실행해봐
주기를 30분으로 바꿔줘
```

각 요청에 대해 에이전트가 SKILL.md의 명령 표를 보고 적절한 `crontab` / `tail` / `bash ... &` 호출을 만들어 실행합니다.

> **이게 OpenClaw 생태계의 핵심 가치입니다.** 자연어로 매뉴얼을 작성하면 에이전트가 그 매뉴얼을 따른다 — 코드 없이 도구를 만들었습니다.
{: .important }

---

## 트러블슈팅

### `env: 'bash\r': No such file or directory`

Windows 줄바꿈(CRLF)이 스크립트에 섞인 것. WSL Ubuntu 안에서 직접 `nano` 로 작성하는 게 가장 안전. 이미 발생했다면:

```bash
sed -i 's/\r$//' ~/.openclaw/scripts/spam-filter.sh
```

### 스크립트가 멈춘 채로 응답 없음

`openclaw infer model run` 호출이 Gateway 통신을 대기 중일 가능성. **Gateway가 떠 있는지 먼저 확인**:

```bash
ps aux | grep "openclaw gateway" | grep -v grep
```

안 떠 있으면 별도 터미널에서 `openclaw gateway` 실행. cron 자동 실행 시점에도 Gateway가 떠 있어야 정상 동작합니다.

### `Model override "..." is not allowed for agent "main"`

스크립트가 `--model <provider/model>` 로 모델을 명시적으로 지정해 메인 에이전트의 허용 모델과 충돌. 스크립트에서 `--model` 옵션을 빼고 OpenClaw 라우팅에 위임하세요. 현재 본문 스크립트는 이미 그렇게 작성돼 있습니다.

### `himalaya envelope list` 가 0건을 돌려줌

- `ACCOUNT` 변수 값이 `himalaya account list` 의 실제 이름과 일치하는지
- 폴더 이름이 `INBOX` 가 맞는지 (`himalaya folder list -a <ACCOUNT>` 로 확인)
- `--output json` 은 **글로벌 옵션** — `himalaya --output json envelope list ...` 순서로 와야 함 (서브명령 뒤 `-O json` 이 아닙니다)

### cron 등록은 했는데 정시에 발화 안 됨

```bash
systemctl status cron
```

`active (running)` 인지 확인. 죽어 있으면:

```bash
sudo systemctl start cron
sudo systemctl enable cron
```

발화 자체 추적:

```bash
journalctl -u cron --since "1 hour ago" | grep spam-filter
```

### Gateway가 새 SKILL.md를 못 봄

Gateway는 시작 시 SKILL을 로드합니다. 변경 후 반드시 재시작:

```bash
systemctl --user restart openclaw-gateway
```

`openclaw skills info linux-cron` 출력이 `△ Needs setup` 이면 SKILL.md frontmatter YAML 문법 오류 — 다른 스킬(`himalaya`) 형식과 비교.

### 에이전트가 옛 cron 컨텍스트로 응답

OpenClaw 워크스페이스 메모리(`~/.openclaw/...` 아래 MEMORY.md 등)에 옛 운영 지식이 박혀 있을 수 있어요. 찾아서 갱신:

```bash
grep -rln "openclaw cron list" ~/.openclaw/ 2>/dev/null
```

발견된 파일의 해당 항목을 새 정책(linux-cron / Ubuntu crontab)으로 수정.

---

## Day 3 산출물

- [ ] `~/.openclaw/scripts/spam-filter.sh` 작성 + 수동 실행 통과 (10통 분류 + JSONL 누적)
- [ ] Ubuntu `crontab -e` 등록 + 다음 정시 자동 발화 검증
- [ ] `linux-cron` SKILL.md 작성 + `Visible to model: yes` 확인
- [ ] 자연어 "스케줄 확인" 으로 에이전트가 `crontab -l` 결과를 표로 응답
- [ ] 자연어 "지금 한번 실행해봐" 로 수동 트리거 가능

다음: [Day 4: Discord 연결 + 대화형 비서](04-day4-discord.html) — 같은 자동화를 Discord 채널로 받고 봇 대화로 운영합니다.
