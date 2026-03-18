## 사전 준비사항

- n8n 설치 (클라우드 또는 셀프호스팅)
- Anthropic 계정 (Claude Code 인증용)
- 연동할 서비스의 API 키 또는 인증 정보 (Google, Slack 등)

---

## Claude Code 설치

### 1단계: VS Code 설치

[Visual Studio Code](https://code.visualstudio.com/) 공식 사이트에서 운영체제에 맞는 버전을 다운로드하여 설치합니다. 코드를 직접 짜는 것이 아니라 Claude Code에게 명령을 내리는 용도로 사용합니다.

### 2단계: Node.js 설치

[Node.js](https://nodejs.org/) 공식 사이트에서 **LTS 버전**을 다운로드하여 설치합니다. Claude Code가 작동하는 데 필요한 런타임 환경입니다.

### 3단계: Claude Code 설치

**방법 1: VS Code Extension (간편)**
1. VS Code 실행
2. 왼쪽 Extensions 탭 클릭
3. "Claude" 검색 → Claude Code extension Install
4. Anthropic 계정으로 로그인하여 인증 완료

**방법 2: 터미널 설치 (커스텀 설정 가능, 추천)**

```bash
# Mac
curl -fsSL https://claude.ai/install.sh | bash

# Windows
irm https://claude.ai/install.ps1 | iex
```

터미널 버전은 더 가볍고, 모든 커스텀 설정이 가능합니다.

### 4단계: 권한 설정 (선택)

Claude Code는 파일을 생성하거나 수정할 때마다 승인을 요청합니다. 이 과정이 번거로울 경우 Settings에서 **"Allow Dangerously Skip Permissions"** 옵션을 활성화할 수 있습니다.

```claude --dangerously-skip-permissions```

> ⚠️ 이 옵션을 활성화하면 Claude Code가 승인 없이 파일을 수정할 수 있으므로, 중요한 프로젝트에서는 주의해서 사용하세요.

`Shift + Tab`으로 다양한 모드(Auto, Plan 등)를 전환하며 사용할 수 있습니다.

---

## n8n MCP & Skills 설치

### MCP와 Skills란?

Claude Code는 범용 AI 도구이기 때문에, n8n 워크플로우를 잘 만들려면 **n8n 전문가 역량**을 장착시켜줘야 합니다.

- **MCP (Model Context Protocol)**: Claude Code가 n8n과 직접 소통할 수 있게 해주는 연결 도구
- **Skills**: n8n 워크플로우를 어떻게 만들어야 하는지 알려주는 지식/스킬셋

이 두 가지를 설치하면 Claude Code가 n8n 워크플로우를 더 정확하고 똑똑하게 만들어줍니다.

### 설치 방법

1. VS Code에서 새 폴더를 생성합니다 (예: `n8n-automation`)
2. 터미널을 열고 해당 디렉토리 안에서 `claude`를 입력하여 Claude Code를 실행합니다
3. 다음 프롬프트를 입력합니다:

```
n8n MCP와 n8n Skills를 이 프로젝트에 설치해줘.

참고링크:
- https://github.com/czlonkowski/n8n-mcp
- https://github.com/czlonkowski/n8n-skills
```

Claude Code가 알아서 필요한 설정을 진행합니다.

### n8n API 연결

MCP가 n8n과 소통하려면 API 키가 필요합니다.

1. n8n에서 **Settings > API**로 이동하여 API 키를 발급
2. MCP 설정에 API 키 입력

**보안 설정 (settings.json):**

```bash
# settings.json 열기
code ~/.claude/settings.json
```

```json
{
  "permissions": {
    "deny": ["Read(.mcp.json)", "Read(**/.mcp.json)"]
  }
}
```

이 설정은 `.mcp.json` 파일 읽기를 제한하여 API 키 등 민감 정보를 보호합니다.

