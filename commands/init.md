---
name: init
description: Initialize workflow-manager plugin - check MCP connections and configure external apps (Slack, Jira, Confluence, GitHub, Calendar)
---

# Workflow Manager 초기 설정

외부 앱 연동 상태를 확인하고 필요한 설정을 진행합니다.

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
→ 성공: cloudId, siteUrl 표시
→ 실패: 설치 가이드 안내
```

#### Slack 테스트
```
mcp__slack__channels_list
→ 성공: 채널 수 표시
→ 실패: 설치 가이드 안내
```

#### Google Calendar 테스트
```
mcp__google-calendar__list-calendars
→ 성공: 캘린더 목록 표시
→ 실패: 설치 가이드 안내 (선택사항)
```

#### GitHub 테스트
```
mcp__plugin_github_github__search_repositories (query: "user:@me")
→ 성공: 레포 수 표시
→ 실패: 설치 가이드 안내 (선택사항)
```

### 3. 결과 출력

연결 상태를 테이블로 표시:

```markdown
## MCP 연결 상태

| 서비스 | 상태 | 정보 |
|--------|------|------|
| Atlassian | ✅ 연결됨 | site: xxx.atlassian.net |
| Slack | ✅ 연결됨 | 채널 N개 |
| Google Calendar | ✅ 연결됨 | 캘린더 N개 |
| GitHub | ❌ 미연결 | - |
```

### 4. 미연결 서비스 설정 안내

연결되지 않은 서비스에 대해 설정 방법을 안내합니다.

#### Atlassian MCP 설치
```
Atlassian MCP가 연결되지 않았습니다.

설치 방법:
1. Claude Code 설정에서 Atlassian MCP 활성화
2. Atlassian 계정으로 인증
3. Claude Code 재시작

또는 수동 설치:
https://github.com/anthropics/claude-code/tree/main/plugins/atlassian
```

#### Slack MCP 설치
```
Slack MCP가 연결되지 않았습니다.

설치 방법:
1. Claude Code 설정에서 Slack MCP 활성화
2. Slack 워크스페이스 연결
3. Claude Code 재시작

또는 수동 설치:
https://github.com/anthropics/claude-code/tree/main/plugins/slack
```

#### Google Calendar MCP 설치
```
Google Calendar MCP가 연결되지 않았습니다. (선택사항)

설치 방법:
1. Claude Code 설정에서 Google Calendar MCP 활성화
2. Google 계정으로 인증
3. Claude Code 재시작
```

#### GitHub MCP 설치
```
GitHub MCP가 연결되지 않았습니다. (선택사항)

설치 방법:
1. GitHub Personal Access Token 생성
   https://github.com/settings/tokens/new
   권한: repo, read:org, read:user

2. 환경변수 설정:
   echo 'export GITHUB_PERSONAL_ACCESS_TOKEN="your-token"' >> ~/.zshrc
   source ~/.zshrc

3. Claude Code 재시작
```

### 5. 기본 설정 구성 (AskUserQuestion)

모든 필수 MCP가 연결된 경우:

```
AskUserQuestion:
  question: "기본 설정을 구성하시겠습니까?"
  header: "설정"
  multiSelect: false
  options:
    - label: "설정 진행 (Recommended)"
      description: "Confluence 스페이스, Jira 프로젝트 기본값 설정"
    - label: "건너뛰기"
      description: "나중에 /workflow setup으로 설정"
```

### 6. Confluence 기본 스페이스 선택

**MCP 도구:** `mcp__atlassian__getConfluenceSpaces`

```
siteId: {cloudId}
```

스페이스 목록을 보여주고 선택:

```
AskUserQuestion:
  question: "작업 문서를 생성할 기본 Confluence 스페이스를 선택하세요"
  header: "스페이스"
  multiSelect: false
  options:
    - label: "{space_name_1}"
      description: "Key: {space_key_1}"
    - label: "{space_name_2}"
      description: "Key: {space_key_2}"
    ...
```

### 7. Confluence 상위 페이지 선택 (선택)

선택된 스페이스의 페이지 목록에서 상위 페이지 선택:

**MCP 도구:** `mcp__atlassian__getPagesInConfluenceSpace`

```
siteId: {cloudId}
spaceId: {selected_space_id}
```

```
AskUserQuestion:
  question: "작업 문서의 상위 페이지를 선택하세요 (선택사항)"
  header: "상위 페이지"
  multiSelect: false
  options:
    - label: "루트에 생성 (Recommended)"
      description: "스페이스 최상위에 문서 생성"
    - label: "{page_title_1}"
      description: "이 페이지 하위에 생성"
    ...
```

### 8. 설정 파일 생성

선택한 내용으로 `~/.claude/workflow.json` 생성:

```json
{
  "confluence": {
    "defaultSpaceKey": "{selected_space_key}",
    "defaultSpaceId": "{selected_space_id}",
    "defaultParentPageId": "{selected_parent_page_id or null}"
  },
  "options": {
    "lookbackDays": 7
  },
  "mcp": {
    "atlassian": {
      "cloudId": "{cloudId}",
      "siteUrl": "{siteUrl}"
    }
  }
}
```

### 9. 완료 메시지

```markdown
## 설정 완료

✅ workflow.json이 생성되었습니다.

### 설정 내용
- **Confluence 스페이스:** {space_name} ({space_key})
- **상위 페이지:** {parent_page_title or "루트"}
- **멘션 검색 기간:** 최근 7일

### 사용 방법
- `/workflow` - 전체 워크플로우 실행
- `/workflow check` - 멘션 확인
- `/workflow todo` - TODO 문서 생성
- `/workflow setup` - 설정 변경

설정 파일: ~/.claude/workflow.json
```

## 에러 처리

### 필수 MCP 미연결 시

Atlassian 또는 Slack이 연결되지 않은 경우:

```
⚠️ 필수 MCP가 연결되지 않았습니다.

미연결:
- Atlassian (필수)
- Slack (필수)

위 서비스를 먼저 연결한 후 다시 `/workflow init`을 실행하세요.
```

### MCP 호출 실패 시

MCP 도구 호출이 실패한 경우:

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
