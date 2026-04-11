---
name: work-log
description: Use when logging work to Supabase, invoked manually via /work-log or automatically via SessionEnd hook. Triggers on session summary, daily report preparation, or work tracking needs.
---

# Work Log

현재 세션의 작업 내용을 요약하여 Supabase에 저장한다. 일일/주간 보고서 작성용.

## 실행 흐름

1. 환경변수 확인
2. 프로젝트 식별
3. 작업 요약 작성
4. 중복 체크
5. 저장

## Step 1: 환경변수 확인

`SUPABASE_URL`과 `SUPABASE_KEY` 환경변수를 확인한다.

없으면 사용자에게 질문:
- "Supabase URL을 입력해주세요 (예: https://xxx.supabase.co)"
- "Supabase anon key를 입력해주세요"

입력받은 값을 `~/.claude/settings.json`의 `env` 필드에 머지하여 저장:

```bash
# 기존 settings.json 읽기
SETTINGS=$(cat ~/.claude/settings.json 2>/dev/null || echo '{}')
# jq로 env 필드에 머지
echo "$SETTINGS" | jq --arg url "$URL" --arg key "$KEY" '.env.SUPABASE_URL = $url | .env.SUPABASE_KEY = $key' > ~/.claude/settings.json
```

## Step 2: 프로젝트 식별

```bash
git remote get-url origin 2>/dev/null
```

결과에서 `owner/repo` 추출:
- HTTPS: `https://github.com/owner/repo.git` → `owner/repo`
- HTTPS (no .git): `https://github.com/owner/repo` → `owner/repo`
- SSH: `git@github.com:owner/repo.git` → `owner/repo`
- GitLab/Bitbucket 등도 동일 패턴

`origin`이 없으면 첫 번째 remote 사용. remote가 아예 없으면 사용자에게 프로젝트명 질문.

## Step 3: 작업 요약 작성

현재 세션에서 수행한 작업을 분석하여 다음을 결정:

- **category**: 다음 중 하나 (영어 고정)
  - `Feature`, `Bugfix`, `Refactor`, `Docs`, `Test`, `Chore`, `Research`, `Review`
- **summary**: 4-5줄 분량의 작업 요약 (**항상 한국어**)
- **user_id**: `git config user.email` 결과
- **ref_type** (선택): 관련 참조가 있으면 `pr`, `issue`, `commit` 중 하나
- **ref_number** (선택): 참조 번호 (예: PR #42 → `ref_type='pr', ref_number=42`)

## Step 4: 중복 체크

같은 프로젝트 + user_id의 최근 24시간 로그를 조회한다.

**타임스탬프**: 현재 시각에서 24시간을 빼서 ISO 8601 형식으로 직접 계산한다. `date` 명령은 OS별로 다르므로 사용하지 않는다.

```bash
curl -s -X GET "${SUPABASE_URL}/rest/v1/work_logs?project=eq.<URL_ENCODED_PROJECT>&user_id=eq.<URL_ENCODED_USER_ID>&created_at=gte.<24H_AGO_ISO>&select=summary,category&order=created_at.desc&limit=20" \
  -H "apikey: ${SUPABASE_KEY}" \
  -H "Authorization: Bearer ${SUPABASE_KEY}"
```

> URL 인코딩: `/` → `%2F`, `@` → `%40`

조회된 기존 로그의 summary와 새 summary를 의미적으로 비교:
- **동일한 작업**: 저장 스킵, 사용자에게 "이미 유사한 로그가 있어 저장을 건너뜁니다" 알림
- **다른 작업**: 정상 저장 진행

## Step 5: 저장

```bash
curl -s -o /tmp/work-log-response.json -w "%{http_code}" -X POST "${SUPABASE_URL}/rest/v1/work_logs" \
  -H "apikey: ${SUPABASE_KEY}" \
  -H "Authorization: Bearer ${SUPABASE_KEY}" \
  -H "Content-Type: application/json" \
  -H "Prefer: return=representation" \
  -d '{"project":"<PROJECT>","category":"<CATEGORY>","summary":"<SUMMARY>","user_id":"<USER_ID>","ref_type":"<REF_TYPE_OR_NULL>","ref_number":<REF_NUMBER_OR_NULL>}'
```

- HTTP 2xx: 저장 성공. 사용자에게 저장된 내용 요약 표시
- HTTP 4xx/5xx: 에러 메시지와 HTTP 상태 코드 표시
- curl 자체 실패 (네트워크 오류 등): "네트워크 오류로 저장에 실패했습니다" 알림. 재시도하지 않음.

## 사전 요구사항

- `jq` 필요 (훅 스크립트에서 JSON 파싱에 사용). 설치 확인:

```bash
command -v jq >/dev/null 2>&1 || echo "jq가 설치되어 있지 않습니다. brew install jq 또는 apt install jq로 설치해주세요."
```

## 첫 실행 시 추가 설정

스킬 최초 실행 시 아래도 수행:

1. `jq` 설치 여부 확인 → 미설치 시 설치 안내
2. `~/.claude/hooks/work-log-session-end.sh`가 없으면 아래 내용으로 생성 + `chmod +x`:

```bash
#!/bin/bash
INPUT=$(cat)
TRANSCRIPT_PATH=$(echo "$INPUT" | jq -r '.transcript_path')
CWD=$(echo "$INPUT" | jq -r '.cwd')

if [[ -f "$TRANSCRIPT_PATH" ]]; then
  CONTEXT=$(jq -c 'select(.role == "user" or .role == "assistant")' "$TRANSCRIPT_PATH" | tail -50)
  cd "$CWD" || exit 0
  echo "$CONTEXT" | claude -p \
    --allowedTools "Bash(curl:*),Bash(git:*)" \
    --max-turns 3 \
    "Based on the session transcript provided on stdin, create a concise work log entry and save it to Supabase. Use the work-log skill instructions: extract project from git remote, check for duplicates, then insert via curl. Summary must be in Korean."
fi
exit 0
```

3. `~/.claude/settings.json`의 `hooks.SessionEnd`에 훅 미등록이면 등록 (기존 설정과 머지):

```json
{"matcher": "", "hooks": [{"type": "command", "command": "~/.claude/hooks/work-log-session-end.sh"}]}
```
