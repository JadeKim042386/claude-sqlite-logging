# work-log

Claude Code 작업 로그를 Supabase에 자동 저장하는 스킬. 일일/주간 보고서 작성용.

## 기능

- `/work-log` 슬래시 커맨드로 수동 저장
- `SessionEnd` 훅으로 세션 종료 시 자동 저장
- 의미적 유사도 기반 중복 방지
- 작업 요약은 항상 한국어, 카테고리는 영어 고정

## 저장 필드

| 필드 | 설명 |
|------|------|
| `project` | git remote에서 추출한 `owner/repo` |
| `category` | Feature, Bugfix, Refactor, Docs, Test, Chore, Research, Review |
| `summary` | 4-5줄 한국어 작업 요약 |
| `user_id` | `git config user.email` |
| `ref_type` | PR/Issue/Commit 참조 (선택) |
| `ref_number` | 참조 번호 (선택) |

## 설치

### 1. 스킬 클론

```bash
git clone https://github.com/JadeKim042386/claude-supabase-logging.git ~/.claude/skills/work-log
```

### 2. Supabase 테이블 생성

Supabase SQL Editor에서 실행:

```sql
CREATE TABLE work_logs (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  project TEXT NOT NULL,
  category TEXT NOT NULL CHECK (category IN (
    'Feature', 'Bugfix', 'Refactor', 'Docs', 'Test', 'Chore', 'Research', 'Review'
  )),
  summary TEXT NOT NULL,
  user_id TEXT NOT NULL,
  ref_type TEXT CHECK (ref_type IN ('pr', 'issue', 'commit')),
  ref_number INT,
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_work_logs_project_user ON work_logs(project, user_id, created_at DESC);
CREATE INDEX idx_work_logs_user_created ON work_logs(user_id, created_at DESC);
CREATE INDEX idx_work_logs_created_at ON work_logs(created_at);
```

### 3. 첫 실행

Claude Code에서 `/work-log`를 실행하면:

1. `SUPABASE_URL`과 `SUPABASE_KEY`를 대화형으로 입력 요청 → `~/.claude/settings.json`에 저장
2. SessionEnd 훅 스크립트 자동 생성 (`~/.claude/hooks/work-log-session-end.sh`)
3. 훅 자동 등록

### 사전 요구사항

- `jq` 설치 필요: `brew install jq` (macOS) / `apt install jq` (Linux)

## Supabase 설정 확인 위치

**Settings → API**에서:
- **Project URL** → `SUPABASE_URL`
- **Project API keys → anon public** → `SUPABASE_KEY`
