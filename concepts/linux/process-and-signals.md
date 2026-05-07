# 프로세스와 시그널

## 한 줄 정의
**프로세스** = 실행 중인 프로그램의 인스턴스 (PID로 식별).
**시그널** = OS가 프로세스에게 보내는 비동기 알림 (예: "종료해라", "설정 다시 읽어라").

## 왜 필요한가
- 시스템에서 **무엇이 돌고 있는지 관측** (monitor.sh의 핵심 역할)
- 죽은 프로세스 / 응답 없는 프로세스 식별
- **정상 종료 vs 강제 종료** 구분 (B1-2 트러블슈팅의 기초)
- 디버깅: 프로세스가 왜 죽었는지

## 핵심 원리

### 프로세스의 정체
```
프로세스 = PID + PPID(부모) + UID/GID + 메모리 + 열린 파일들 + 환경 변수
```

- **PID** (Process ID): 정수, 1부터 시작. PID 1은 init/systemd
- **PPID** (Parent PID): 누가 나를 fork했는지
- 모든 프로세스는 부모를 가짐 (PID 1만 예외)

### 프로세스 생성: fork-exec
```
부모 프로세스 (셸)
    │
    ├─ fork() → 자식 프로세스 복제 (메모리 그대로)
    │
    └─ 자식: exec("/usr/bin/python") → 메모리 새 프로그램으로 교체
```

→ 모든 명령은 셸이 fork하고 exec하는 방식. 
→ 환경 변수는 fork 시점에 자식에게 복사됨 (그래서 export가 필요).

### 프로세스 상태 (ps의 STAT 컬럼)
```
R  Running       실행 중 또는 실행 가능 (큐 대기)
S  Sleeping      이벤트 대기, 인터럽트 가능 (대부분의 idle)
D  Disk sleep    인터럽트 불가, 보통 I/O 대기
Z  Zombie        죽었는데 부모가 회수 안 함
T  Stopped       Ctrl+Z 등으로 일시 중단
```

### 시그널: 비동기 알림

| 번호 | 이름 | 의미 | 잡을 수 있나? |
|---|---|---|---|
| 1  | SIGHUP  | 터미널 끊김 / "설정 재읽기" 관습 | ✅ |
| 2  | SIGINT  | Ctrl+C, 사용자 인터럽트 | ✅ |
| 9  | **SIGKILL** | **즉시 종료** (cleanup 없음) | ❌ |
| 15 | **SIGTERM** | **정상 종료 요청** (cleanup 후 끝내라) | ✅ (기본 동작은 종료) |
| 17 | SIGCHLD | 자식 프로세스 죽었음 알림 | ✅ |
| 19 | SIGSTOP | 일시 정지 | ❌ |
| 20 | SIGTSTP | Ctrl+Z (일시 정지 요청) | ✅ |

### SIGTERM vs SIGKILL — 가장 중요

```
SIGTERM (15): "끝낼 시간이야, 정리하고 끝내"
              → 프로세스가 try/finally로 cleanup 가능
              → 임시 파일 삭제, 연결 종료, 상태 저장
              → 그래도 안 끝나면 그때 SIGKILL

SIGKILL (9):  "즉시 죽어" — OS가 강제 종료
              → cleanup 코드 절대 실행 안 됨
              → 임시 파일 남음, DB 트랜잭션 깨짐 가능
              → "최후의 수단"
```

`kill -9 PID`가 위험한 이유: cleanup 못 하니까 **데이터 손상 위험**.
정석: `kill PID` (SIGTERM) 먼저 → 안 되면 `kill -9 PID`.

### 좀비와 고아
- **좀비(Z)**: 자식이 죽었는데 부모가 `wait()`으로 회수 안 함. PID만 차지하고 메모리 거의 없음.
- **고아**: 부모가 먼저 죽은 자식 → PID 1(init)이 입양 → init이 정상 회수.

## 자주 쓰는 명령

```bash
# 프로세스 보기
ps aux                       # 모든 프로세스 (BSD 스타일, 가장 흔함)
ps -ef                       # 모든 프로세스 (System V 스타일)
ps aux | grep agent_app      # 특정 프로세스 찾기
ps -p 1234 -o %cpu,%mem      # 특정 PID의 CPU/MEM만

# ps aux 컬럼:
# USER  PID  %CPU  %MEM  VSZ  RSS  TTY  STAT  START  TIME  COMMAND
#                              └─ 실제 메모리(KB)
#                        └─ 가상 메모리(KB)

# PID 찾기
pgrep -f agent_app           # 패턴(전체 명령줄)으로
pgrep agent_app              # 이름으로 (정확히)
pidof agent_app.py           # 정확한 이름으로

# 시그널 보내기
kill 1234                    # SIGTERM (15) — 정상 종료 요청
kill -15 1234                # 명시적 SIGTERM
kill -9 1234                 # SIGKILL — 강제
kill -HUP 1234               # SIGHUP — 보통 "재로드"
kill -l                      # 모든 시그널 목록

# 패턴으로 죽이기 (조심!)
pkill -f agent_app           # 매칭되는 모든 프로세스에 SIGTERM
killall agent_app.py         # 정확한 이름의 모든 인스턴스에

# 실시간 관측
top                          # 대화형 모니터링
top -b -n 1                  # 1회 출력 후 종료 (스크립트용 ★)
htop                         # 더 보기 좋은 top (별도 설치)
```

## 흔한 오해

- ❌ "kill 명령은 죽이는 명령" → ✅ 이름은 그렇지만 실제는 **시그널 전송 도구**. SIGUSR1(사용자 정의)도 보낼 수 있음.
- ❌ "kill -9가 깔끔하다" → ✅ **반대**. cleanup 못 해서 더 더러움. **SIGTERM 먼저, 안 되면 SIGKILL**.
- ❌ "프로세스 응답 없으면 좀비" → ✅ 응답 없는 건 D 또는 S 상태일 수 있음. 좀비(Z)는 "이미 죽었지만 회수 안 됨".
- ❌ "SIGINT(Ctrl+C)와 SIGTERM은 같다" → ✅ 둘 다 잡을 수 있고 기본 동작이 종료라는 점은 같지만, 구분되는 시그널. 스크립트는 다르게 처리할 수 있음.
- ❌ "ps aux의 %CPU가 그 순간 사용률" → ✅ 보통 **시작 후 평균** (구현마다 다름). 순간값은 `top`이 더 정확.

## 이번 과제(B1-1)에서

monitor.sh의 health check 부분:
```bash
# 1. 프로세스 살아있나?
PID=$(pgrep -f agent_app.py)
if [ -z "$PID" ]; then
    echo "[ERROR] agent_app.py not running"
    exit 1
fi

# 2. 그 프로세스의 CPU/MEM 측정
ps -p $PID -o %cpu=,%mem= --no-headers
```

cron이 monitor.sh를 매분 실행:
- cron이 자식 프로세스로 monitor.sh를 fork+exec
- 끝나면 cron이 정상 회수 → 좀비 안 생김

앱 종료가 Ctrl+C라는 건 → SIGINT를 받음 → 앱이 잡아서 정상 종료 처리.

## 관련 개념
- [filesystem-tree.md](./filesystem-tree.md) — `/proc/<PID>/`로 프로세스 정보 노출
- [shell-environment.md](./shell-environment.md) — 환경 변수는 fork 시 복사
- (예정) `concepts/os/cpu-measurement.md` — `/proc/stat`로 CPU 사용률 정확히 계산
- (예정) B1-2 [oom-and-watchdog.md] — 시그널 기반 종료의 실제 사례

## 출처
- B1-1 (Layer 1.5)
- 학습일: 2026-05-07
- 참고: `man 7 signal`, `man ps`, `man kill`, `man proc`
