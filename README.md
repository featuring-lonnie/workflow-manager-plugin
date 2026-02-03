# Workflow Manager Plugin

Claude Code 플러그인으로 Slack, Confluence, Jira, GitHub, Google Calendar를 통합하여 작업을 관리합니다.

## 기능

- **멘션 수집**: Slack과 Confluence에서 나를 멘션한 내용 자동 수집
- **TODO 추출**: 멘션에서 작업 요청 패턴을 분석하여 TODO 목록 생성
- **작업 계획**: Confluence에 작업 계획 문서 자동 생성
- **Jira 연동**: 작업에 대한 Jira 티켓 자동 생성 및 상태 관리
- **캘린더 연동**: 작업 블록을 Google Calendar에 생성

## 요구사항

다음 MCP 서버가 연결되어 있어야 합니다:

- **Atlassian MCP** - Confluence, Jira 연동
- **Slack MCP** - Slack 멘션 검색
- **Google Calendar MCP** - 캘린더 일정 관리
- **GitHub MCP** (선택) - PR 정보 연동

## 설치

```bash
# 마켓플레이스 추가
/plugin marketplace add featuring-lonnie/lonnie-marketplace

# 플러그인 설치
/plugin install workflow-manager@lonnie-marketplace
```

## 사용법

### 커맨드

| 커맨드 | 설명 |
|--------|------|
| `/workflow-manager` | **명령어 메뉴** - 사용 가능한 명령어 안내 |
| `/workflow-manager init` | 초기 설정 - MCP 연결 확인 및 외부 앱 설정 (최초 1회) |
| `/workflow-manager run` | **전체 워크플로우 실행** (수집 → 분석 → 계획 → 실행) |
| `/workflow-manager check` | 새 멘션만 확인 (티켓 생성 안 함) |
| `/workflow-manager todo` | Confluence + Slack 멘션에서 TODO 추출 → 개인 문서 생성 |
| `/workflow-manager complete` | 현재 작업 완료 처리 |
| `/workflow-manager setup` | MCP 연결 확인 및 기본값 설정 |
| `/workflow-manager status` | 진행 중인 작업 상태 확인 |

### 워크플로우

```
/workflow-manager run 실행
    │
    ▼
Phase 1: 수집 (Collect)
├─ Slack 멘션 검색
├─ 스레드 컨텍스트 수집
├─ Confluence 멘션 검색
└─ 캘린더 일정 확인
    │
    ▼
Phase 2: 분석 (Analyze)
├─ 작업 요청 패턴 식별
├─ TODO 목록 생성
└─ 사용자에게 TODO 제시 및 선택 요청
    │
    ▼
Phase 3: 계획 (Plan)
├─ Jira 프로젝트 자동 매핑
└─ Confluence 작업 계획 문서 생성
    │
    ▼
Phase 4: 실행 (Execute)
├─ Jira 티켓 생성
├─ Confluence 문서에 Jira 링크 추가
└─ (선택) 캘린더 작업 블록 생성
    │
    ▼
Phase 5: 완료 (/workflow-manager complete)
├─ Confluence 문서 업데이트
└─ Jira 티켓 상태 전환
```

## 설정 (선택)

`~/.claude/workflow.json`으로 기본값 설정 가능:

```json
{
  "confluence": {
    "defaultSpaceKey": "MYSPACE",
    "defaultParentPageId": "123456789"
  },
  "options": {
    "lookbackDays": 7
  }
}
```

## License

MIT
