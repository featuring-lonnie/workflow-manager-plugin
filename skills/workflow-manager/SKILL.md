---
name: workflow-manager
description: Use when managing work tasks across Slack mentions, Confluence documents, Jira tickets, and GitHub PRs. Triggers on /workflow command to collect mentions, create TODOs, generate work plans, and sync progress across platforms.
---

# Workflow Manager

통합 작업 관리 워크플로우 - Slack, Confluence, Jira, GitHub, Google Calendar를 연동하여 작업을 수집하고 관리합니다.

## MCP 연동

별도 설정 파일 없이 MCP 서버에서 직접 정보를 조회합니다.

### 시작 시 조회할 정보

1. **Atlassian 리소스**
   ```
   mcp__atlassian__getAccessibleAtlassianResources
   → cloudId, siteUrl 획득
   ```

2. **현재 사용자 정보**
   ```
   mcp__atlassian__atlassianUserInfo
   → accountId, displayName, email 획득
   ```

3. **Slack 채널 목록** (멘션 검색용)
   ```
   mcp__slack__channels_list
   → 채널 ID 목록 획득
   ```

4. **Google Calendar 목록**
   ```
   mcp__google-calendar__list-calendars
   → calendarId, account 획득
   ```

### 선택적 설정 (옵션)

`~/.claude/workflow.json`으로 기본값 오버라이드 가능:

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

## 서브 커맨드

| 커맨드 | 설명 |
|--------|------|
| `/workflow` | 전체 워크플로우 실행 (수집 → 분석 → 계획 → 실행) |
| `/workflow check` | 새 멘션만 확인 (티켓 생성 안 함) |
| `/workflow todo` | Confluence + Slack 멘션에서 TODO 추출 → 선택 → 개인 문서 생성 |
| `/workflow complete` | 현재 작업 완료 처리 (Jira 전환, Confluence 업데이트) |
| `/workflow setup` | MCP 연결 확인 및 선택적 기본값 설정 |
| `/workflow status` | 진행 중인 작업 상태 확인 |

## 워크플로우 개요

```
/workflow 실행
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
Phase 3: 계획 (Plan) - 선택 시
├─ Git remote → Jira 프로젝트 자동 매핑
├─ 매핑 없으면 사용자에게 질문
└─ Confluence 작업 계획 문서 생성
    │
    ▼
Phase 4: 실행 (Execute) - 승인 시
├─ Jira 티켓 생성
├─ Confluence 문서에 Jira 링크 추가
└─ (선택) 캘린더 작업 블록 생성
    │
    ▼
Phase 5: 완료 (/workflow complete)
├─ Confluence 문서 업데이트
└─ Jira 티켓 상태 전환
```

---

## Phase 1: 수집 (Collect)

### 1.1 MCP 연동 정보 조회

**Atlassian 리소스 조회:**
```
mcp__atlassian__getAccessibleAtlassianResources
→ cloudId 획득
```

**현재 사용자 정보 조회:**
```
mcp__atlassian__atlassianUserInfo
→ accountId (Confluence 멘션 검색용)
```

**선택적 설정 파일 확인:**
```
Read ~/.claude/workflow.json (없으면 기본값 사용)
```

### 1.2 Slack 멘션 검색

**MCP 도구:** `mcp__slack__conversations_search_messages`

```
query: "to:me" 또는 사용자 이름으로 검색
```

최근 7일간 (또는 설정된 lookbackDays) 멘션 검색.

검색 결과에서 각 메시지의:
- `channel`: 채널 ID
- `ts`: 타임스탬프
- `text`: 메시지 내용
- `user`: 보낸 사람

### 1.3 스레드 컨텍스트 수집

멘션된 메시지가 스레드의 일부인 경우:

**MCP 도구:** `mcp__slack__conversations_replies`

```
channel: {channel_id}
ts: {thread_ts}
```

스레드 전체 컨텍스트를 수집하여 작업 요청의 맥락 파악

### 1.4 Confluence 멘션 검색

**MCP 도구:** `mcp__atlassian__searchConfluenceUsingCql`

```
cql: "mention = currentUser() AND lastmodified > now('-{lookbackDays}d')"
siteId: {atlassian.cloudId}
```

### 1.5 캘린더 일정 확인

**MCP 도구:** `mcp__google-calendar__list-events`

```
calendarId: {calendar.id}
account: {calendar.account}
timeMin: (오늘)
timeMax: (오늘 + 7일)
```

기존 일정과 충돌하지 않도록 확인

---

## Phase 2: 분석 (Analyze)

### 2.1 작업 요청 패턴 식별

수집된 멘션에서 다음 패턴을 찾습니다:

**명시적 요청:**
- "해주세요", "부탁드립니다", "확인해주세요"
- "please", "could you", "can you"
- 물음표로 끝나는 질문

**암시적 요청:**
- 버그 리포트 형태
- 리뷰 요청
- 피드백 요청

### 2.2 TODO 목록 생성

각 작업 요청을 다음 형식으로 정리:

```markdown
## 수집된 작업 요청

### [1] {제목}
- **출처:** Slack #{channel_name} | Confluence {page_title}
- **요청자:** @{username}
- **일시:** {timestamp}
- **내용:** {요약된 요청 내용}
- **스레드 컨텍스트:** {스레드가 있으면 요약}

### [2] {제목}
...
```

### 2.3 사용자 확인

**AskUserQuestion 사용:**

```
어떤 작업을 진행하시겠습니까?

[ ] 1. {작업 제목 1}
[ ] 2. {작업 제목 2}
[ ] 3. {작업 제목 3}
[ ] 모두 선택
[ ] 건너뛰기
```

---

## Phase 3: 계획 (Plan)

### 3.1 Jira 프로젝트 매핑

**현재 Git remote 확인:**
```bash
git remote get-url origin
```

**projectMapping에서 매칭:**
```
remote URL → projectMapping → Jira Project Key
```

매핑이 없으면:

**MCP 도구:** `mcp__atlassian__getVisibleJiraProjects`

```
siteId: {atlassian.cloudId}
```

사용자에게 프로젝트 선택 요청

### 3.2 Confluence 문서 미리보기 및 컨펌 (필수)

**문서 생성 전 반드시 사용자에게 전체 내용을 보여주고 피드백을 받습니다.**

마크다운으로 다음 내용을 출력:

```markdown
## Confluence 문서 미리보기

**Space:** {space_key}
**상위 페이지:** {parent_page_title}
**제목:** [WIP] {작업 제목} - {날짜}

### 문서 내용

{문서 전체 내용 마크다운으로 표시}

---
위 내용으로 Confluence 문서를 생성하시겠습니까?
```

**AskUserQuestion 사용:**

```
옵션:
1. 승인 - 위 내용으로 문서 생성
2. 수정 요청 - 내용 수정 후 다시 확인
3. 취소 - 문서 생성 안 함
```

**수정 요청 시:**
- 사용자 피드백을 반영하여 내용 수정
- 다시 미리보기 표시
- 승인될 때까지 반복

### 3.3 Confluence 문서 생성 (승인 후)

**MCP 도구:** `mcp__atlassian__createConfluencePage`

```
siteId: {atlassian.cloudId}
spaceId: {confluence.spaceId}
parentPageId: {confluence.parentPageId}
title: "[WIP] {작업 제목} - {날짜}"
bodyFormat: "atlas_doc_format"
bodyValue: (아래 템플릿 참조)
```

> **참고:** `bodyFormat: "atlas_doc_format"`을 사용하면 Confluence 라이브 문서(Live Doc)로 생성됩니다.

**Confluence 문서 템플릿:**

```html
<h1>작업 개요</h1>
<table>
  <tr><th>항목</th><th>내용</th></tr>
  <tr><td>작업명</td><td>{작업 제목}</td></tr>
  <tr><td>요청자</td><td><ac:link><ri:user ri:account-id="{accountId}"/></ac:link></td></tr>
  <tr><td>요청일</td><td>{요청 날짜}</td></tr>
  <tr><td>Jira 티켓</td><td>(실행 단계에서 추가)</td></tr>
  <tr><td>상태</td><td><ac:structured-macro ac:name="status"><ac:parameter ac:name="colour">Blue</ac:parameter><ac:parameter ac:name="title">진행 중</ac:parameter></ac:structured-macro></td></tr>
</table>

<h2>요청 내용</h2>
<p>{원본 요청 내용}</p>

<h3>출처</h3>
<ul>
  <li>Slack: <a href="{slack_link}">#{channel_name}</a></li>
</ul>

<h2>구현 계획</h2>
<ol>
  <li>{계획 단계 1}</li>
  <li>{계획 단계 2}</li>
  <li>{계획 단계 3}</li>
</ol>

<h2>완료 기준</h2>
<ul>
  <li>{완료 기준 1}</li>
  <li>{완료 기준 2}</li>
</ul>

<h2>진행 상황</h2>
<p>(작업 진행 시 업데이트)</p>

<hr/>
<p><em>🤖 Created by Claude Code</em></p>
```

### 3.4 다음 단계 확인

**AskUserQuestion 사용:**

```
작업 계획이 생성되었습니다.

Confluence: {page_url}

다음 단계로 진행하시겠습니까?
- Jira 티켓 생성
- (선택) 캘린더 작업 블록 생성

[승인] [수정 요청] [취소]
```

---

## Phase 4: 실행 (Execute)

### 4.0 티켓 내용 미리보기 및 피드백 (필수)

**티켓 생성 전 반드시 사용자에게 전체 내용을 보여주고 피드백을 받습니다.**

마크다운으로 다음 내용을 출력:

```markdown
## Jira 티켓 미리보기

**프로젝트:** {project_key}
**유형:** Task
**제목:** {작업 제목}

### 설명

{description 전체 내용}

### 완료 기준
- [ ] {완료 기준 1}
- [ ] {완료 기준 2}

### 관련 링크
- Slack 스레드: [{slack_channel_name}]({slack_permalink})

---
위 내용으로 티켓을 생성하시겠습니까?
```

**AskUserQuestion 사용:**

```
옵션:
1. 승인 - 위 내용으로 티켓 생성
2. 수정 요청 - 내용 수정 후 다시 확인
3. 취소 - 티켓 생성 안 함
```

**수정 요청 시:**
- 사용자 피드백을 반영하여 내용 수정
- 다시 미리보기 표시
- 승인될 때까지 반복

### 4.1 Jira 티켓 생성 (승인 후)

**요청자 Account ID 조회:**

**MCP 도구:** `mcp__atlassian__lookupJiraAccountId`

```
siteId: {atlassian.cloudId}
query: {요청자 이메일 또는 이름}
```

**티켓 생성:**

**MCP 도구:** `mcp__atlassian__createJiraIssue`

```
siteId: {atlassian.cloudId}
projectKey: {mapped_project_key}
issueType: {atlassian.defaultIssueType}
summary: {작업 제목}
description: (아래 템플릿 참조)
```

**Jira 티켓 Description 템플릿:**

```markdown
## 요청 내용

{원본 요청 내용}

## 관련 링크

* Confluence 작업 계획: [{confluence_page_title}]({confluence_page_url})
* Slack 스레드: [{slack_channel_name}]({slack_permalink})

## 완료 기준

* {완료 기준 1}
* {완료 기준 2}

----
🤖 _Created by Claude Code_
```

### 4.2 Confluence 문서 업데이트

**MCP 도구:** `mcp__atlassian__updateConfluencePage`

Jira 티켓 링크 추가:

```html
<tr><td>Jira 티켓</td><td><a href="{jira_issue_url}">{issue_key}</a></td></tr>
```

### 4.3 캘린더 작업 블록 생성 (선택)

**MCP 도구:** `mcp__google-calendar__create-event`

```
calendarId: {calendar.id}
account: {calendar.account}
summary: "[작업] {작업 제목}"
description: "Jira: {issue_key}\nConfluence: {page_url}"
start: (사용자 지정 또는 다음 가용 시간)
end: (start + 2시간)
```

---

## Phase 5: 완료 (/workflow complete)

### 5.1 현재 작업 확인

현재 디렉토리의 git 정보로 진행 중인 작업 식별:
- Branch 이름에서 Jira 티켓 키 추출 (예: `feature/PROJ-123-description`)
- 또는 사용자에게 완료할 작업 선택 요청

### 5.2 PR 정보 수집

```bash
gh pr view --json url,title,number
```

### 5.3 Confluence 문서 업데이트

**MCP 도구:** `mcp__atlassian__getConfluencePage` (현재 내용 조회)
**MCP 도구:** `mcp__atlassian__updateConfluencePage`

업데이트 내용:
- 상태: "진행 중" → "완료"
- PR 링크 추가
- 진행 상황에 완료 내용 추가

```html
<tr><td>상태</td><td><ac:structured-macro ac:name="status"><ac:parameter ac:name="colour">Green</ac:parameter><ac:parameter ac:name="title">완료</ac:parameter></ac:structured-macro></td></tr>

<h2>결과물</h2>
<ul>
  <li>PR: <a href="{pr_url}">{pr_title}</a></li>
</ul>
```

### 5.4 Jira 티켓 상태 전환

**MCP 도구:** `mcp__atlassian__getTransitionsForJiraIssue`

```
siteId: {atlassian.cloudId}
issueKey: {issue_key}
```

가능한 전환 중 "Done", "완료", "Resolved" 등 찾기

**MCP 도구:** `mcp__atlassian__transitionJiraIssue`

```
siteId: {atlassian.cloudId}
issueKey: {issue_key}
transitionId: {done_transition_id}
```

---

## /workflow check

멘션만 확인하고 티켓 생성은 하지 않습니다.

1. Phase 1 (수집) 실행
2. Phase 2 (분석) 실행
3. TODO 목록만 표시하고 종료

---

## /workflow todo

Confluence와 Slack에서 멘션된 내용을 수집하여 TODO 항목을 추출하고, 개인 문서로 정리합니다.

### 단계별 워크플로우

#### 1. Confluence 멘션 검색

**MCP 도구:** `mcp__atlassian__searchConfluenceUsingCql`

```
siteId: {atlassian.cloudId}
cql: "mention = currentUser() AND lastmodified > now('-7d')"
limit: 25
```

#### 2. Slack 멘션 검색

나를 직접 멘션하거나, 내가 속한 그룹(@here, @channel, 사용자 그룹)을 멘션한 메시지를 검색합니다.

**MCP 도구:** `mcp__slack__conversations_search_messages`

```
query: "to:me"
limit: 50
```

> **참고:** `to:me`는 나를 직접 멘션한 메시지와 내가 속한 그룹 멘션을 모두 포함합니다.

#### 3. 스레드 컨텍스트 수집 (필요 시)

Slack 메시지가 스레드의 일부인 경우 전체 컨텍스트를 수집:

**MCP 도구:** `mcp__slack__conversations_replies`

```
channel: {channel_id}
ts: {thread_ts}
```

#### 4. Confluence 페이지 내용 분석

각 Confluence 검색 결과에서:

**MCP 도구:** `mcp__atlassian__getConfluencePage`

```
siteId: {atlassian.cloudId}
pageId: {page_id}
includeBody: true
bodyFormat: "storage"
```

#### 5. TODO 항목 추출

**Confluence에서:**
- 직접적인 작업 요청 ("@{user} 확인 부탁드립니다", "@{user} 처리해주세요")
- 체크리스트 항목 (`<ac:task>` 태그)
- 마감일이 언급된 요청
- 질문이나 피드백 요청

**Slack에서:**
- 작업 요청 패턴 ("확인해주세요", "처리 부탁", "리뷰 요청", "검토 필요")
- 질문 패턴 ("?", "어떻게", "언제", "가능할까요")
- 마감일 언급 ("오늘까지", "내일까지", "이번 주")
- 긴급 표시 ("긴급", "ASAP", "급함")

#### 6. TODO 목록 선택 (AskUserQuestion)

추출된 TODO를 사용자에게 제시하고 선택받기:

```
AskUserQuestion:
  question: "다음 TODO 항목 중 문서에 포함할 항목을 선택하세요"
  header: "TODO 선택"
  multiSelect: true
  options:
    - label: "{todo_1_title}"
      description: "📄 Confluence: {page_title}"
    - label: "{todo_2_title}"
      description: "💬 Slack: #{channel_name}"
    - label: "{todo_3_title}"
      description: "📄 Confluence: {page_title}"
    ...
```

#### 5. 문서 생성 위치 확인 (AskUserQuestion)

```
AskUserQuestion:
  question: "TODO 문서를 어디에 생성할까요?"
  header: "위치 선택"
  multiSelect: false
  options:
    - label: "개인 스페이스 (Recommended)"
      description: "내 개인 Confluence 스페이스에 생성"
    - label: "특정 스페이스 지정"
      description: "다른 스페이스를 선택"
```

#### 6. 문서 미리보기 및 확인 (AskUserQuestion)

생성할 문서 내용을 먼저 보여주고 확인:

```
## 생성할 문서 미리보기

**제목:** {date} TODO
**위치:** {space_name}

### 내용:
{document_preview}

---
🤖 Created by Claude Code
```

```
AskUserQuestion:
  question: "위 내용으로 문서를 생성할까요?"
  header: "문서 확인"
  multiSelect: false
  options:
    - label: "생성 (Recommended)"
      description: "위 내용으로 TODO 문서 생성"
    - label: "수정 필요"
      description: "내용을 수정하고 다시 확인"
    - label: "취소"
      description: "문서 생성 취소"
```

#### 7. TODO 문서 생성

**MCP 도구:** `mcp__atlassian__createConfluencePage`

```
siteId: {atlassian.cloudId}
spaceId: {personal_space_id}
title: "{YYYY-MM-DD} TODO"
status: "current"
parentPageId: null (또는 지정된 상위 페이지)
bodyFormat: "atlas_doc_format"
bodyValue: (아래 템플릿 참조)
```

> **참고:** `bodyFormat: "atlas_doc_format"`을 사용하면 Confluence 라이브 문서(Live Doc)로 생성됩니다.

### TODO 문서 템플릿 (Storage Format)

```xml
<ac:structured-macro ac:name="info">
  <ac:rich-text-body>
    <p>Confluence와 Slack 멘션에서 추출된 TODO 목록입니다.</p>
  </ac:rich-text-body>
</ac:structured-macro>

<h2>📋 TODO 목록</h2>

<ac:task-list>
  <!-- Confluence 출처 TODO -->
  <ac:task>
    <ac:task-id>{unique_id_1}</ac:task-id>
    <ac:task-status>incomplete</ac:task-status>
    <ac:task-body>
      <p><strong>{todo_title}</strong></p>
      <p>📄 출처: <ac:link><ri:page ri:content-title="{source_page_title}" /></ac:link></p>
    </ac:task-body>
  </ac:task>

  <!-- Slack 출처 TODO -->
  <ac:task>
    <ac:task-id>{unique_id_2}</ac:task-id>
    <ac:task-status>incomplete</ac:task-status>
    <ac:task-body>
      <p><strong>{todo_title}</strong></p>
      <p>💬 출처: <a href="{slack_permalink}">#{channel_name}</a></p>
    </ac:task-body>
  </ac:task>

  <!-- 선택된 TODO 항목들 반복 -->
</ac:task-list>

<hr />
<p><em>🤖 Created by Claude Code</em></p>
```

---

## /workflow setup

MCP 연결 상태를 확인하고, 선택적 기본값을 설정합니다.

### 단계별 안내:

1. **MCP 연결 확인**

   **Atlassian 연결:**
   ```
   mcp__atlassian__getAccessibleAtlassianResources
   mcp__atlassian__atlassianUserInfo
   ```

   **Slack 연결:**
   ```
   mcp__slack__channels_list
   ```

   **Google Calendar 연결:**
   ```
   mcp__google-calendar__list-calendars
   ```

2. **Confluence Space 선택 (선택적 기본값)**

   **MCP 도구:** `mcp__atlassian__getConfluenceSpaces`

   - 사용 가능한 Space 목록 표시
   - 작업 문서를 생성할 기본 Space 선택

3. **Confluence Parent Page 선택 (선택적)**

   **MCP 도구:** `mcp__atlassian__getPagesInConfluenceSpace`

   - 선택된 Space의 페이지 목록 표시
   - 작업 문서를 생성할 상위 페이지 선택

4. **선택적 설정 파일 생성**

   기본값을 저장하고 싶으면 `~/.claude/workflow.json` 생성:
   ```json
   {
     "confluence": {
       "defaultSpaceKey": "선택한_SPACE_KEY",
       "defaultParentPageId": "선택한_PAGE_ID"
     }
   }
   ```

---

## /workflow status

진행 중인 작업 상태를 확인합니다.

1. 현재 git branch에서 Jira 키 추출
2. Jira 티켓 정보 조회
3. 관련 Confluence 문서 조회
4. 상태 요약 표시:

```
## 현재 작업 상태

**Jira:** PROJ-123 - {제목}
**상태:** In Progress
**Confluence:** {page_url}
**브랜치:** feature/PROJ-123-description

### 최근 활동
- {jira_comment_1}
- {jira_comment_2}
```

---

## 에러 처리

### MCP 도구 실패 시

1. 에러 메시지를 사용자에게 표시
2. 가능한 원인 안내:
   - 인증 만료: "MCP 서버 재연결이 필요할 수 있습니다"
   - 권한 부족: "해당 리소스에 대한 권한을 확인하세요"
   - 네트워크 오류: "잠시 후 다시 시도하세요"
3. 워크플로우를 안전하게 중단하거나 해당 단계 건너뛰기

### MCP 연결 실패 시

MCP 서버가 연결되지 않으면:
```
MCP 연결을 확인할 수 없습니다.

실패한 서비스:
- Atlassian: 연결 안 됨
- Slack: 연결 안 됨

MCP 서버 설정을 확인하세요.
`/workflow setup`으로 연결 상태를 점검할 수 있습니다.
```

### 매핑 실패 시

Git remote → Jira 프로젝트 매핑이 없으면:
```
이 저장소에 대한 Jira 프로젝트 매핑이 없습니다.

현재 저장소: github.com/company/repo

사용할 Jira 프로젝트를 선택하세요:
1. PROJ - Project Name
2. DEV - Development
3. 직접 입력
```

---

## 주의사항

- **미리보기 필수:** Jira 티켓, Confluence 문서 생성 전에 반드시 전체 내용을 미리보기로 보여주고 사용자 승인을 받을 것
- **Claude Code 표시:** 모든 생성물에 `🤖 Created by Claude Code` 표시 포함
- **멱등성:** 같은 멘션에 대해 중복 티켓이 생성되지 않도록 확인
- **권한:** 각 서비스의 권한 범위 내에서만 작업
- **컨텍스트 유지:** 각 Phase에서 수집한 정보를 다음 Phase로 전달
- **사용자 확인:** 중요한 작업(티켓 생성, 상태 전환) 전에 항상 사용자 확인
