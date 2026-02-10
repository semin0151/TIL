# Claude Code 알림 설정

## 개요
Claude Code 응답 완료 시 macOS 알림이 뜨도록 설정함.

## 사용 도구
- **terminal-notifier**: Homebrew로 설치된 macOS 알림 유틸리티

## 설정 방법

### 1. terminal-notifier 설치
```bash
brew install terminal-notifier
```

### 2. Stop Hook 등록
`~/.claude/settings.json`에 Stop hook 추가:
```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "terminal-notifier -title 'Claude Code' -message 'Claude가 응답을 완료했습니다' -sender com.claude.notifier"
          }
        ]
      }
    ]
  }
}
```

### 3. 옵션 설명
| 옵션 | 설명 |
|------|------|
| `-title` | 알림 제목 |
| `-message` | 알림 본문 |
| `-sender` | 알림 아이콘으로 사용할 앱의 번들 ID |

## 참고
- `-sender com.claude.notifier`로 Claude 아이콘 적용
- `-sound` 옵션을 제거하여 무음 알림으로 설정
- `-sender com.mitchellh.ghostty` 같은 옵션은 앱 탐색 과정에서 지연이 발생할 수 있으므로 주의
