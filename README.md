# KakaoCLI Windows

Windows 카카오톡(x64)의 대화 DB를 복호화하고 조회하는 독립 실행 도구입니다.

## 파일 구성

* `kakaocli-win-gui.exe`: 더블클릭하여 화면에서 [지금 갱신]을 누르는 GUI 프로그램
* `kakaocli-win.exe`: 터미널(PowerShell/CMD) 및 외부 프로그램(Python, Node.js 등) 연동용 CLI 프로그램

---

## 1. 사전 요구사항

* Windows 10/11 64비트(x64)
* 카카오톡 PC버전 실행 및 로그인 상태

---

## 2. CLI 명령어 상세 사용법 (`kakaocli-win.exe`)

모든 데이터 조회 명령은 `--json` 플래그를 지원하며, 표준 출력(stdout)으로 JSON 데이터를 반환합니다.

### 2.1 환경 및 버전 진단 (`doctor`)
카카오톡 프로세스 실행 여부, x64 아키텍처, 빌드 버전 및 지원 여부를 진단합니다.

```powershell
.\kakaocli-win.exe doctor
```

* **출력 JSON 예시**:
```json
{
  "platform": "windows",
  "architecture": "x86_64",
  "implementation": "rust-native",
  "kakao_running": true,
  "process_id": 23300,
  "executable": "C:\\Program Files\\Kakao\\KakaoTalk\\KakaoTalk.exe",
  "executable_version": "26.8.2.5324",
  "supported": true
}
```

---

### 2.2 대화 DB 복호화 갱신 (`refresh`)
실행 중인 카카오톡 메모리에서 암호화 키를 읽어와, 로컬 SQLite 복사본으로 복호화하고 WAL을 병합합니다. **대화를 조회하기 전 최소 1회 실행해야 합니다.**

```powershell
.\kakaocli-win.exe refresh --json
```

* **주요 옵션**:
  * `--json`: 결과를 JSON 형식으로 출력
  * `--no-remote-thread`: 백신/보안 프로그램으로 인해 원격 스레드 주입이 차단될 때 읽기 전용 힙 메모리 탐색 모드로 실행
  * `--data-dir <DIR>`: 복호화된 DB를 저장할 출력 디렉터리 지정 (기본값: `%LOCALAPPDATA%\kakaocli`)

* **출력 JSON 예시**:
```json
{
  "status": "success",
  "chats_recovered": 185,
  "wal_frames_merged": 320,
  "integrity_check": "ok"
}
```

---

### 2.3 대화방 목록 조회 (`chats`)
최근 대화가 오간 대화방 목록을 조회합니다.

```powershell
.\kakaocli-win.exe chats --limit 20 --json
```

* **주요 옵션**:
  * `--limit <N>`: 조회할 대화방 개수 (기본값: 20)
  * `--json`: JSON 형식 출력

* **반환 필드 규약**:
  * `id` (number): 순번
  * `chat_id` (number): 대화방 고유 ID (64비트 정수)
  * `type` (string): 방 종류 (`"direct"`: 1:1, `"group"`: 그룹채팅, `"open"`: 오픈채팅, `"memo"`: 나와의 채팅)
  * `display_name` (string): 대화방 이름
  * `member_count` (number): 참여자 수
  * `unread_count` (number): 안 읽은 메시지 수
  * `last_message_at` (number): 마지막 메시지 수신 시각 (초 단위 Unix Timestamp)

* **출력 JSON 예시**:
```json
[
  {
    "id": 1,
    "chat_id": 18273918273,
    "type": "group",
    "display_name": "프로젝트 개발팀",
    "member_count": 4,
    "unread_count": 0,
    "last_message_at": 1700000000
  }
]
```

---

### 2.4 메시지 내역 조회 (`messages`)
특정 대화방의 메시지 내역을 조회합니다.

```powershell
# 1) 방 이름(substring 매칭)으로 조회
.\kakaocli-win.exe messages --chat "프로젝트 개발팀" --limit 50 --json

# 2) chat_id로 정확히 조회
.\kakaocli-win.exe messages --chat-id 18273918273 --limit 50 --json

# 3) 최근 N시간/N일 전 메시지만 조회
.\kakaocli-win.exe messages --chat "프로젝트 개발팀" --since 24h --json
```

* **주요 옵션**:
  * `--chat <NAME>`: 대화방 이름 (부분 일치 검색)
  * `--chat-id <ID>`: 대화방 고유 ID (정확한 64비트 정수)
  * `--since <DURATION>`: 특정 시점 이후 조회 (`10m`, `1h`, `2d` 등)
  * `--limit <N>`: 가져올 최대 메시지 수 (기본값: 50)
  * `--json`: JSON 형식 출력

* **반환 필드 규약**:
  * `id` (number): 메시지 고유 로그 ID
  * `chat_id` (number): 소속 대화방 ID
  * `sender_id` (number): 발신자 고유 계정 ID
  * `sender` (string): 발신자 이름
  * `text` (string): 메시지 본문
  * `type` (number): 메시지 타입 (`1`: 텍스트, `2`: 이미지, `12`: 이모티콘, `26`: 파일 등)
  * `timestamp` (string): ISO-8601 형식의 발송 시각 (`"2026-09-22T10:30:00Z"`)
  * `is_from_me` (boolean): 내가 보낸 메시지 여부 (`true` / `false`)

* **출력 JSON 예시**:
```json
[
  {
    "id": 10521,
    "chat_id": 18273918273,
    "sender_id": 987654321,
    "sender": "홍길동",
    "text": "오늘 회의 자료 공유드립니다.",
    "type": 1,
    "timestamp": "2026-09-22T10:30:00Z",
    "is_from_me": false
  }
]
```

---

### 2.5 대화 내용 검색 (`search`)
모든 대화방의 전체 메시지에서 키워드를 검색합니다.

```powershell
.\kakaocli-win.exe search "회의" --limit 20 --json
```

* **주요 옵션**:
  * `<QUERY>`: 검색할 단어 또는 문장
  * `--limit <N>`: 검색 결과 최대 개수 (기본값: 20)
  * `--json`: JSON 형식 출력

---

## 3. Python 연동 코드 예제

외부 스크립트에서 `kakaocli-win.exe`를 서브프로세스로 호출해 데이터를 다루는 표준 예제입니다:

```python
import subprocess
import json
from pathlib import Path

CLI = Path(".\\kakaocli-win.exe")

def run_cmd(args):
    proc = subprocess.run([str(CLI), *args, "--json"], capture_output=True, text=True, encoding="utf-8")
    if proc.returncode != 0:
        raise RuntimeError(f"CLI Error: {proc.stderr}")
    return json.loads(proc.stdout)

# 1. DB 갱신
run_cmd(["refresh"])

# 2. 대화방 목록 가져오기
chats = run_cmd(["chats", "--limit", "10"])
for c in chats:
    print(f"[{c['chat_id']}] {c['display_name']} ({c['member_count']}명)")

# 3. 특정 대화방의 최근 30개 메시지 가져오기
if chats:
    target_id = str(chats[0]["chat_id"])
    messages = run_cmd(["messages", "--chat-id", target_id, "--limit", "30"])
    for m in messages:
        sender = "나" if m["is_from_me"] else m["sender"]
        print(f"[{m['timestamp']}] {sender}: {m['text']}")
```

---

## 4. 복호화된 SQLite DB 직접 쿼리 명세

`refresh` 완료 후 생성되는 표준 SQLite 파일에 직접 접근하여 SQL 쿼리를 실행할 수도 있습니다.

* **DB 저장 경로**: `%LOCALAPPDATA%\kakaocli\recovered_sqlite\`
  * `core/chatListInfo.edb`: 대화방 목록 메타데이터
  * `chat_data/<XX>/<CHAT_ID>.edb`: 대화방별 메시지 내역

### 테이블 스키마:
1. **`chatRoomList` 테이블** (`core/chatListInfo.edb`):
   * `chatId` (INTEGER, PK): 대화방 고유 번호
   * `type` (TEXT): `"direct"`, `"group"`, `"open"`, `"memo"`
   * `chatRoomTitle` (TEXT): 대화방 이름
   * `activeMembersCount` (INTEGER): 참여 멤버 수
   * `lastUpdatedAt` (INTEGER): 마지막 메시지 수신 시각 (Unix Timestamp)

2. **`chatLogs` 테이블** (`chat_data/<XX>/<CHAT_ID>.edb`):
   * `id` / `logId` (INTEGER, PK): 메시지 고유 번호
   * `chatId` (INTEGER): 소속 대화방 ID
   * `authorId` (INTEGER): 작성자 카카오 계정 ID
   * `message` (TEXT): 메시지 본문
   * `type` (INTEGER): 메시지 타입 코드
   * `createdAt` (INTEGER): 발송 시각 (초 단위 Unix Timestamp)
