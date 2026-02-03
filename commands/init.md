---
name: init
description: Initialize workflow-manager plugin - check MCP connections and configure external apps (Slack, Jira, Confluence, GitHub, Calendar)
---

# Workflow Manager 초기 설정

외부 앱 연동 상태를 확인하고 주요 채널/스페이스/프로젝트를 설정합니다.

## 실행 단계

### 1. MCP 연결 상태 확인

다음 MCP 서버들의 연결 상태를 확인합니다:

**필수:**
- Atlassian (Jira, Confluence)
- Slack

**선택:**
- Google Calendar
- GitHub

### 2. 각 MCP 테스트

#### Atlassian 테스트
```
mcp__atlassian__getAccessibleAtlassianResources
→ 성공: cloudId, siteUrl 획득
→ 실패: 설치 가이드 안내
```

#### Slack 테스트
```
mcp__slack__channels_list
→ 성공: 채널 목록 획득
→ 실패: 설치 가이드 안내
```

#### Google Calendar 테스트
```
mcp__google-calendar__list-calendars
→ 성공: 캘린더 목록 획득
→ 실패: 설치 가이드 안내 (선택사항)
```

#### GitHub 테스트
```
mcp__plugin_github_github__search_repositories (query: "user:@me")
→ 성공: 레포 목록 획득
→ 실패: 설치 가이드 안내 (선택사항)
```

### 3. 연결 상태 출력

```markdown
## MCP 연결 상태

| 서비스 | 상태 | 정보 |
|--------|------|------|
| Atlassian | ✅ 연결됨 | site: xxx.atlassian.net |
| Slack | ✅ 연결됨 | 채널 N개 접근 가능 |
| Google Calendar | ✅ 연결됨 | 캘린더 N개 |
| GitHub | ❌ 미연결 | - |
```

### 4. 미연결 서비스 설정 안내

(기존과 동일 - 설치 가이드 제공)

---

## 앱별 상세 설정

### 5. Slack 설정

#### 5.1 주요 채널 선택

**MCP 도구:** `mcp__slack__channels_list`

멘션을 모니터링할 주요 채널을 선택합니다:

```
AskUserQuestion:
  question: "멘션을 모니터링할 주요 Slack 채널을 선택하세요"
  header: "Slack 채널"
  multiSelect: true
  options:
    - label: "#{channel_name_1}"
      description: "{channel_purpose or 'No description'}"
    - label: "#{channel_name_2}"
      description: "{channel_purpose}"
    - label: "#{channel_name_3}"
      description: "{channel_purpose}"
    - label: "전체 채널"
      description: "모든 채널에서 멘션 검색"
```

#### 5.2 DM 포함 여부

```
AskUserQuestion:
  question: "DM(다이렉트 메시지)도 포함할까요?"
  header: "DM 설정"
  multiSelect: false
  options:
    - label: "포함 (Recommended)"
      description: "DM에서 받은 요청도 수집"
    - label: "제외"
      description: "채널 멘션만 수집"
```

---

### 6. Confluence 설정

#### 6.1 기본 스페이스 선택

**MCP 도구:** `mcp__atlassian__getConfluenceSpaces`

```
AskUserQuestion:
  question: "작업 문서를 생성할 기본 Confluence 스페이스를 선택하세요"
  header: "기본 스페이스"
  multiSelect: false
  options:
    - label: "{space_name_1}"
      description: "Key: {space_key_1}"
    - label: "{space_name_2}"
      description: "Key: {space_key_2}"
```

#### 6.2 모니터링할 스페이스 선택

멘션을 검색할 스페이스를 선택합니다:

```
AskUserQuestion:
  question: "멘션을 모니터링할 Confluence 스페이스를 선택하세요"
  header: "모니터링 스페이스"
  multiSelect: true
  options:
    - label: "{space_name_1}"
      description: "Key: {space_key_1}"
    - label: "{space_name_2}"
      description: "Key: {space_key_2}"
    - label: "전체 스페이스"
      description: "모든 스페이스에서 멘션 검색"
```

#### 6.3 상위 페이지 선택 (선택)

**MCP 도구:** `mcp__atlassian__getPagesInConfluenceSpace`

```
AskUserQuestion:
  question: "작업 문서의 상위 페이지를 선택하세요"
  header: "상위 페이지"
  multiSelect: false
  options:
    - label: "루트에 생성 (Recommended)"
      description: "스페이스 최상위에 문서 생성"
    - label: "{page_title_1}"
      description: "이 페이지 하위에 생성"
```

---

### 7. Jira 설정

#### 7.1 기본 프로젝트 선택

**MCP 도구:** `mcp__atlassian__getVisibleJiraProjects`

```
AskUserQuestion:
  question: "티켓을 생성할 기본 Jira 프로젝트를 선택하세요"
  header: "기본 프로젝트"
  multiSelect: false
  options:
    - label: "{project_name_1}"
      description: "Key: {project_key_1}"
    - label: "{project_name_2}"
      description: "Key: {project_key_2}"
```

#### 7.2 Git 저장소 매핑

현재 작업 중인 Git 저장소와 Jira 프로젝트를 연결합니다:

```bash
git remote get-url origin
→ github.com/company/repo-name
```

```
AskUserQuestion:
  question: "이 저장소({repo_name})와 연결할 Jira 프로젝트를 선택하세요"
  header: "저장소 매핑"
  multiSelect: false
  options:
    - label: "{project_name_1} (Recommended)"
      description: "Key: {project_key_1}"
    - label: "{project_name_2}"
      description: "Key: {project_key_2}"
    - label: "매핑 안 함"
      description: "나중에 수동으로 선택"
```

#### 7.3 기본 이슈 타입

**MCP 도구:** `mcp__atlassian__getJiraProjectIssueTypesMetadata`

```
AskUserQuestion:
  question: "기본 이슈 타입을 선택하세요"
  header: "이슈 타입"
  multiSelect: false
  options:
    - label: "Task (Recommended)"
      description: "일반 작업"
    - label: "Story"
      description: "사용자 스토리"
    - label: "Bug"
      description: "버그 리포트"
```

#### 7.4 모니터링할 프로젝트 (선택)

담당 티켓 알림을 받을 프로젝트:

```
AskUserQuestion:
  question: "담당 티켓 알림을 받을 프로젝트를 선택하세요"
  header: "알림 프로젝트"
  multiSelect: true
  options:
    - label: "{project_name_1}"
      description: "Key: {project_key_1}"
    - label: "{project_name_2}"
      description: "Key: {project_key_2}"
    - label: "전체 프로젝트"
      description: "모든 프로젝트의 담당 티켓"
```

---

### 8. GitHub 설정 (연결된 경우)

#### 8.1 모니터링할 저장소

**MCP 도구:** `mcp__plugin_github_github__search_repositories`

```
AskUserQuestion:
  question: "PR/Issue 알림을 받을 GitHub 저장소를 선택하세요"
  header: "GitHub 저장소"
  multiSelect: true
  options:
    - label: "{repo_name_1}"
      description: "{repo_description_1}"
    - label: "{repo_name_2}"
      description: "{repo_description_2}"
    - label: "전체 저장소"
      description: "모든 저장소의 멘션된 PR/Issue"
```

#### 8.2 PR 리뷰 알림

```
AskUserQuestion:
  question: "PR 리뷰 요청 알림을 받으시겠습니까?"
  header: "PR 리뷰"
  multiSelect: false
  options:
    - label: "예 (Recommended)"
      description: "리뷰 요청된 PR을 TODO에 포함"
    - label: "아니오"
      description: "PR 리뷰 알림 제외"
```

---

### 9. Google Calendar 설정 (연결된 경우)

#### 9.1 기본 캘린더 선택

**MCP 도구:** `mcp__google-calendar__list-calendars`

```
AskUserQuestion:
  question: "작업 블록을 생성할 기본 캘린더를 선택하세요"
  header: "기본 캘린더"
  multiSelect: false
  options:
    - label: "{calendar_name_1} (Recommended)"
      description: "{calendar_id_1}"
    - label: "{calendar_name_2}"
      description: "{calendar_id_2}"
```

#### 9.2 일정 확인용 캘린더

충돌 확인 시 참조할 캘린더:

```
AskUserQuestion:
  question: "일정 충돌 확인 시 참조할 캘린더를 선택하세요"
  header: "참조 캘린더"
  multiSelect: true
  options:
    - label: "{calendar_name_1}"
      description: "Primary"
    - label: "{calendar_name_2}"
      description: "Work"
    - label: "{calendar_name_3}"
      description: "Personal"
```

---

### 10. 추가 옵션 설정

```
AskUserQuestion:
  question: "추가 옵션을 설정하세요"
  header: "옵션"
  multiSelect: true
  options:
    - label: "멘션 검색 기간: 7일 (Recommended)"
      description: "최근 7일간의 멘션 검색"
    - label: "멘션 검색 기간: 14일"
      description: "최근 14일간의 멘션 검색"
    - label: "자동 Jira 티켓 생성 비활성화"
      description: "항상 수동 확인 후 생성"
    - label: "캘린더 작업 블록 자동 생성"
      description: "티켓 생성 시 자동으로 캘린더에 추가"
```

---

### 11. 설정 파일 생성

선택한 내용으로 `~/.claude/workflow.json` 생성:

```json
{
  "slack": {
    "watchChannels": ["C01234567", "C89012345"],
    "includeDM": true
  },
  "confluence": {
    "defaultSpaceKey": "MYSPACE",
    "defaultSpaceId": "123456",
    "defaultParentPageId": null,
    "watchSpaces": ["MYSPACE", "TEAM"]
  },
  "jira": {
    "defaultProjectKey": "PROJ",
    "defaultIssueType": "Task",
    "watchProjects": ["PROJ", "DEV"],
    "repoMapping": {
      "github.com/company/repo": "PROJ"
    }
  },
  "github": {
    "watchRepos": ["company/repo1", "company/repo2"],
    "includePRReviews": true
  },
  "calendar": {
    "defaultCalendarId": "primary",
    "checkConflictCalendars": ["primary", "work@group.calendar.google.com"]
  },
  "options": {
    "lookbackDays": 7,
    "autoCreateTicket": true,
    "autoCreateCalendarBlock": false
  },
  "mcp": {
    "atlassian": {
      "cloudId": "{cloudId}",
      "siteUrl": "{siteUrl}"
    }
  }
}
```

---

### 12. 완료 메시지

```markdown
## 설정 완료

✅ workflow.json이 생성되었습니다.

### Slack
- **모니터링 채널:** #channel1, #channel2
- **DM 포함:** 예

### Confluence
- **기본 스페이스:** My Space (MYSPACE)
- **모니터링 스페이스:** MYSPACE, TEAM

### Jira
- **기본 프로젝트:** Project (PROJ)
- **기본 이슈 타입:** Task
- **저장소 매핑:** github.com/company/repo → PROJ

### GitHub
- **모니터링 저장소:** company/repo1, company/repo2
- **PR 리뷰 알림:** 예

### Calendar
- **기본 캘린더:** Primary
- **충돌 확인:** Primary, Work

### 옵션
- **멘션 검색 기간:** 7일
- **자동 티켓 생성:** 예
- **자동 캘린더 블록:** 아니오

---

### 사용 방법
- `/workflow` - 전체 워크플로우 실행
- `/workflow check` - 멘션 확인
- `/workflow todo` - TODO 문서 생성
- `/workflow setup` - 설정 변경

설정 파일: ~/.claude/workflow.json
```

---

## 에러 처리

### 필수 MCP 미연결 시

```
⚠️ 필수 MCP가 연결되지 않았습니다.

미연결:
- Atlassian (필수)
- Slack (필수)

위 서비스를 먼저 연결한 후 다시 `/workflow init`을 실행하세요.
```

### MCP 호출 실패 시

```
❌ {서비스} 연결 테스트 실패

오류: {error_message}

가능한 원인:
- 인증 만료
- 네트워크 오류
- 권한 부족

해결 방법:
1. Claude Code 재시작
2. 재인증 시도
3. MCP 서버 상태 확인
```
