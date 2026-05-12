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
- **자연어로 작성된 `SKILL.md`가 에이전트의 새 능력이 되는 패턴**을 직접 만들어본다
- OpenClaw cron으로 자동 분류 파이프라인을 등록하고 운영한다

## 시간표

| 시간 | 블록 | 내용 |
|---|---|---|
| 0:00–0:50 | 1 | 스팸 분류 개념 + 안전장치 / 단일 메일 수동 분류·이동 |
| 1:00–1:50 | 2 | 사전 셋업 (권한 패치) / `openclaw-cron` SKILL.md 작성 |
| 2:00–2:50 | 3 | cron 작업 등록 / 자연어로 자동화 관리 |
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

## Block 2: 자동화를 위한 사전 셋업

### 2-1. OpenClaw 권한 패치 (필수)

OpenClaw 2026.5.x에는 **단일 사용자 페어링 시 권한 부족** 회귀 버그가 있습니다 ([Issue #74484](https://github.com/openclaw/openclaw/issues/74484), [#29387](https://github.com/openclaw/openclaw/issues/29387)). 기본 페어링된 CLI device가 `operator.read`만 받기 때문에 cron 등록·관리(`operator.write`·`operator.pairing` 필요)가 막힙니다.

> ⚠️ 이 패치 없이는 `openclaw cron add` 가 "scope upgrade pending approval" 에러로 영원히 막힙니다. 반드시 먼저 적용하세요.
{: .warning }

#### 패치 절차

```bash
# jq 미설치 시
sudo apt install -y jq

# 백업
cp ~/.openclaw/devices/paired.json ~/.openclaw/devices/paired.json.bak

# 모든 권한 부여 — 3곳 모두 (scopes, approvedScopes, tokens.operator.scopes)
SCOPES='["operator.read", "operator.write", "operator.admin", "operator.pairing"]'
jq --argjson scopes "$SCOPES" '
  .[] |= (
    .scopes = $scopes
    | .approvedScopes = $scopes
    | .tokens.operator.scopes = $scopes
  )
' ~/.openclaw/devices/paired.json > /tmp/paired.json
mv /tmp/paired.json ~/.openclaw/devices/paired.json

# 게이트웨이가 덮어쓰지 못하게 readonly 잠금
chmod 444 ~/.openclaw/devices/paired.json

# 게이트웨이 재시작
systemctl --user restart openclaw-gateway
sleep 3

# 검증
openclaw devices list
```

Paired 행의 Scopes 컬럼에 4개 권한이 모두 보여야 정상.

> `chmod 444` 잠금이 핵심입니다. 안 하면 게이트웨이가 다시 `operator.read`만 남기게 덮어씁니다.
{: .important }

### 2-2. `openclaw-cron` SKILL.md 작성

여기가 Day 3의 백미입니다 — **자연어 매뉴얼**로 에이전트의 새 능력을 정의합니다.

```bash
mkdir -p ~/.openclaw/workspace/skills/openclaw-cron
nano ~/.openclaw/workspace/skills/openclaw-cron/SKILL.md
```

다음 내용 붙여넣기 (Ctrl+O, Enter, Ctrl+X로 저장):

```
---
name: openclaw-cron
description: Manage OpenClaw gateway cron jobs — list, show, create, run, disable, enable, delete schedules. Use whenever the user asks about schedules, cron, recurring tasks, automation timing, 스케줄, or background jobs.
metadata:
  openclaw:
    emoji: "⏰"
    requires:
      bins:
        - openclaw
---

# OpenClaw Cron 스킬

OpenClaw 게이트웨이의 cron 레지스트리를 관리한다. MCP 도구로는 노출되지 않으므로 모든 동작은 `openclaw cron` CLI를 Bash로 호출해서 수행한다.

## When to use

사용자가 다음과 같이 요청할 때:
- "스케줄 확인", "등록된 자동화 보여줘", "cron 작업 목록"
- "5분마다 X 해줘", "매시간 Y 실행"
- "방금 자동화 어디까지 동작했어?"
- "이 자동화 멈춰줘", "스케줄 삭제"

## Steps

1. 사용자 의도를 아래 표의 intent 중 하나로 매핑한다.
2. 해당 CLI 명령을 Bash로 실행한다.
3. 결과를 사용자에게 보기 좋은 형식(표)으로 정리해 응답한다.

## CLI 명령 매핑

| intent | CLI |
|---|---|
| 등록된 모든 작업 목록 | openclaw cron list |
| 특정 작업 상세 | openclaw cron show <ID> |
| 새 작업 등록 | openclaw cron add --name <n> --every <duration> --session main --system-event "<프롬프트>" |
| 즉시 실행 (디버그) | openclaw cron run <ID> |
| 실행 이력 | openclaw cron runs --id <ID> --limit 10 |
| 스케줄러 상태 | openclaw cron status |
| 비활성화 / 활성화 | openclaw cron disable <ID> / openclaw cron enable <ID> |
| 삭제 | openclaw cron rm <ID> |

## 응답 포맷

목록·상세는 표 형식: Name, Schedule, Next, Last, Status, ID 컬럼.

작업이 0건이면 "현재 등록된 스케줄 없음"만 답한다.

새 작업 등록 성공 시 Job ID, Name, Schedule, Next run을 보여준다.

## Rules

- 같은 name으로 이미 등록된 작업이 있을 때 사용자가 또 등록 요청하면, 기존 것을 보여주고 덮어쓸지 확인한다.
- delete/rm 전에는 항상 한 번 더 확인을 받는다.
- 새 작업은 기본적으로 --session main으로 (사용자가 메인 세션에서 결과 보길 원할 가능성).

## Constraints

- 반복 cron은 7일 후 자동 만료된다. 갱신 필요 시 사용자에게 안내.

## Don't

- mcp__openclaw__subagents 또는 mcp__openclaw__sessions_list로 cron 상태를 추론하지 말 것. 그 도구들은 cron 레지스트리를 보지 못한다.
- Claude Code 내장 CronCreate/CronList(.claude/scheduled_tasks.json)와 OpenClaw 게이트웨이 cron은 완전히 별개. 사용자가 "스케줄"이라고 하면 기본은 OpenClaw cron 기준.
```

#### 등록 확인

```bash
openclaw skills info openclaw-cron
```

기대 출력:
- `✓ Ready`
- `Visible to model: yes` — 에이전트가 자동으로 이 스킬을 인식·사용 가능
- Requirements: `✓ openclaw`

게이트웨이 캐시 갱신:

```bash
systemctl --user restart openclaw-gateway
sleep 3
```

#### 스킬 동작 검증

OpenClaw TUI 진입:

```bash
openclaw
# Crestodian → talk to agent
```

자연어로 (이전 컨텍스트 없이):

```
스케줄 확인
```

에이전트가 자동으로 `openclaw cron list`를 호출해 표 형식으로 응답해야 정상.

또는:

```
스케줄 관련 명령어 목록 보여줘
```

→ SKILL.md의 명령 매핑이 그대로 표 형태로 출력.

> **이게 OpenClaw 생태계의 핵심 가치입니다.** 자연어로 매뉴얼을 작성하면 에이전트가 그 매뉴얼을 따른다 — 코드 없이 도구를 만들었습니다.
{: .important }

---

## Block 3: cron 자동 분류 등록

### 3-1. 자연어로 등록 (권장)

가장 OpenClaw다운 방식. TUI 메인 에이전트에:

```
spam-filter라는 이름으로 cron 작업 등록해줘. 5분마다 실행.
작업 내용: INBOX 최근 메일 1~3통을 가져와서 spam 분류하고, spam이면 ai-test로 himalaya로 이동, 결과 한 줄 요약.
세션은 main, 시스템 이벤트로 주입.
```

에이전트가 방금 만든 `openclaw-cron` 스킬을 사용해 적절한 `openclaw cron add` 명령을 만들고 실행합니다. 성공 시 Job ID와 함께 등록 결과 출력.

### 3-2. CLI로 직접 등록 (대안)

자연어가 어려우면:

```bash
openclaw cron add \
  --name spam-filter \
  --every 5m \
  --session main \
  --system-event "INBOX 최근 메일 1~3통 spam 분류, spam이면 ai-test로 himalaya로 이동, 결과 한 줄 요약"
```

---

## Block 4: 운영 + 검증

### 등록 확인

```
스케줄 확인
```

`spam-filter` 1건이 every 5m, idle 상태로 보여야 정상.

### 즉시 실행 (5분 기다리기 싫으면)

```
spam-filter를 지금 한 번 실행해줘
```

또는:

```bash
openclaw cron run <JOB_ID>
```

### 발화 이력

```
spam-filter의 최근 실행 이력 5건 보여줘
```

또는:

```bash
openclaw cron runs --id <JOB_ID> --limit 5
```

### 메일 상태 확인

```bash
himalaya envelope list | head -5      # INBOX
himalaya envelope list -f ai-test     # 이동된 메일
```

새로 spam으로 분류·이동된 메일이 있으면 자동화 정상 동작.

### 비활성화 / 삭제

```
spam-filter 잠시 꺼줘    # disable
spam-filter 다시 켜줘    # enable
spam-filter 삭제해줘     # rm (에이전트가 한 번 더 확인 요청)
```

또는 CLI:

```bash
openclaw cron disable <ID>
openclaw cron enable <ID>
openclaw cron rm <ID>
```

---

## 트러블슈팅

### "scope upgrade pending approval" 영원히 막힘

OpenClaw 회귀 버그. [Block 2-1](#2-1-openclaw-권한-패치-필수)의 paired.json 패치를 적용했는지 확인. `chmod 444` 잊지 마세요 — 안 하면 게이트웨이가 다시 덮어씁니다.

### cron 등록은 됐는데 발화 안 됨

```bash
openclaw cron status
```

`enabled: true` 인지 확인. 아니면:

```bash
openclaw config set cron.enabled true
systemctl --user restart openclaw-gateway
```

게이트웨이 자체가 죽었으면 cron도 안 발화:

```bash
systemctl --user status openclaw-gateway
```

### `runs`에 발화는 기록되는데 실제 메일 이동 안 됨

`durationMs`가 1~5ms로 매우 짧다면 cron이 system event만 enqueue하고 즉시 종료한 것. 실제 분류·이동은 메인 세션에서 비동기로 일어나므로:

- TUI를 켜둬야 메인 세션이 깨어 처리
- 또는 cron 프롬프트를 더 명확하게: "반드시 himalaya로 실제 이동까지 수행하고, 작업 끝나면 한 줄 요약"

### LLM이 spam을 ham으로 잘못 분류 (또는 반대)

프롬프트 보강. cron 프롬프트를 다음처럼 명시:

```
오늘 날짜는 <YYYY-MM-DD>이다. 학습 데이터 기준 미래 날짜라고 spam으로 판단하지 말 것.
스팸 판정 기준:
- 발신자가 명백한 정상 서비스 도메인 → ham
- 광고/마케팅/피싱성 → spam
- 본인이 동의한 알림(영수증, 인증) → ham
- 모호하면 ham
```

### 반복 cron이 7일 후 자동 만료됨

OpenClaw cron의 알려진 제약. 갱신:

```
spam-filter를 같은 사양으로 새로 등록해줘
```

### SKILL.md를 만들었는데 에이전트가 인식 못함

`openclaw skills info openclaw-cron` 출력에서 `Visible to model: yes`인지 확인. 만약 `△ Needs setup`이면 SKILL.md frontmatter의 YAML 문법이 잘못된 것. 다른 스킬 (`himalaya`) 형식과 비교.

게이트웨이 재시작 잊었으면:

```bash
systemctl --user restart openclaw-gateway
```

### `AGENTS.md`/`TOOLS.md`는 자동 인식 안 되는데 SKILL.md는 됨

알려진 OpenClaw 버그 ([Issue #29387](https://github.com/openclaw/openclaw/issues/29387)) — bootstrap 파일 자동 주입에 회귀가 있습니다. 워크스페이스 루트의 AGENTS.md/TOOLS.md에 둔 가이드가 자동 로드 안 될 수 있어요. 그 경우 에이전트에 "TOOLS.md 읽어줘"라고 명시적으로 요청하거나, 스킬 형식(`skills/<name>/SKILL.md`)으로 옮기면 안정적입니다.

---

## 부록 (advanced): bash 스크립트 + `openclaw infer` 방식

OpenClaw cron 대신 결정적인 셸 스크립트 + 시스템 cron으로 자동화하고 싶을 때. LLM은 분류만, 셸이 이동·로깅을 직접 수행하므로 LLM hallucinate 영향 없음. 시스템 cron이 동작하면 TUI 안 켜져 있어도 발화.

### 스크립트 생성

```bash
mkdir -p ~/.openclaw/scripts ~/.openclaw/logs
nano ~/.openclaw/scripts/spam-filter.sh
```

내용:

```bash
#!/usr/bin/env bash
set -euo pipefail

TARGET_FOLDER="ai-test"
BATCH_SIZE=5
SLEEP_BETWEEN_CALLS=5
LOG_FILE="$HOME/.openclaw/logs/spam-classify.jsonl"

mkdir -p "$(dirname "$LOG_FILE")"
touch "$LOG_FILE"

mapfile -t IDS < <(himalaya envelope list -o json 2>/dev/null \
    | jq -r '.[].id' | head -"$BATCH_SIZE")

for ID in "${IDS[@]}"; do
  # 멱등성 — 이미 처리한 ID 스킵
  if [ -s "$LOG_FILE" ] && jq -e --argjson id "$ID" \
       'select(.id == $id)' "$LOG_FILE" >/dev/null 2>&1; then
    continue
  fi

  CONTENT=$(himalaya message read --preview "$ID" 2>/dev/null || true)
  [ -z "$CONTENT" ] && continue

  SUBJECT=$(echo "$CONTENT" | grep -m1 '^Subject:' | sed 's/^Subject: //' || true)
  FROM=$(echo "$CONTENT" | grep -m1 '^From:' | sed 's/^From: //' || true)

  PROMPT="Classify the following email as spam or ham. Output STRICTLY raw JSON only.

Email:
$CONTENT

Format: {\"decision\":\"spam|ham\",\"reason\":\"한 줄 설명\"}"

  RAW=$(openclaw infer model run --prompt "$PROMPT" 2>&1 || true)
  JSON=$(echo "$RAW" | grep -oE '\{"decision":[^}]+\}' | head -1 || true)

  if [ -z "$JSON" ]; then
    jq -nc --arg ts "$(date -Iseconds)" --argjson id "$ID" \
      --arg subject "$SUBJECT" --arg from "$FROM" \
      --arg decision "error" --arg reason "LLM no text output" \
      --arg action "skipped" \
      '{ts:$ts,id:$id,subject:$subject,from:$from,decision:$decision,reason:$reason,action:$action}' \
      >> "$LOG_FILE"
    sleep "$SLEEP_BETWEEN_CALLS"
    continue
  fi

  DECISION=$(echo "$JSON" | jq -r .decision)
  REASON=$(echo "$JSON" | jq -r .reason)

  ACTION="kept"
  if [ "$DECISION" = "spam" ]; then
    himalaya message move "$TARGET_FOLDER" "$ID" >/dev/null 2>&1 \
      && ACTION="moved_to_$TARGET_FOLDER" \
      || ACTION="move_failed"
  fi

  jq -nc --arg ts "$(date -Iseconds)" --argjson id "$ID" \
    --arg subject "$SUBJECT" --arg from "$FROM" \
    --arg decision "$DECISION" --arg reason "$REASON" \
    --arg action "$ACTION" \
    '{ts:$ts,id:$id,subject:$subject,from:$from,decision:$decision,reason:$reason,action:$action}' \
    >> "$LOG_FILE"

  sleep "$SLEEP_BETWEEN_CALLS"
done
```

실행 권한 + 시스템 cron 등록:

```bash
chmod +x ~/.openclaw/scripts/spam-filter.sh

# 시스템 crontab (5분마다)
crontab -e
# 다음 한 줄 추가:
# */5 * * * * /home/<user>/.openclaw/scripts/spam-filter.sh
```

### 검증

```bash
# 수동 1회 실행
~/.openclaw/scripts/spam-filter.sh

# 로그 확인
cat ~/.openclaw/logs/spam-classify.jsonl | jq | tail
```

### 트레이드오프

| | 본문 (OpenClaw cron + 스킬) | 부록 (bash + infer) |
|---|---|---|
| 학습 부담 | 낮음 (자연어로 관리) | 높음 (bash·jq·cron 표현식) |
| TUI 의존 | 발화 시 TUI 켜져 있어야 처리 | 무관 |
| LLM hallucinate 영향 | 있음 (메인 세션이 처리) | 없음 (셸이 결정적) |
| 강의 정체성 일치 | 높음 (OpenClaw 생태계) | 낮음 (OpenClaw 외부) |

---

## Day 3 산출물

- [ ] `paired.json` scope 패치 + readonly 잠금 적용
- [ ] `openclaw-cron` SKILL.md 작성 + 에이전트 자동 인식 확인 (`Visible to model: yes`)
- [ ] 자연어 "스케줄 확인"으로 cron 상태 조회 가능
- [ ] `spam-filter` cron 작업 등록 + 5분 후 자동 발화 검증
- [ ] 메일 상태(INBOX·ai-test) 변화 관찰

다음: Day 4: Discord 연결 + 대화형 비서 *(작성 예정)*
