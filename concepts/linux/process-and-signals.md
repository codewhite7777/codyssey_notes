# 프로세스 모델과 시그널

## 개요
Linux의 프로세스는 PID로 식별되는 실행 컨텍스트로, `fork()`로 복제 + `execve()`로 메모리 교체를 통해 생성된다. **시그널**은 OS가 프로세스에 전달하는 비동기 알림 메커니즘으로, 종료·재시작·재구성 등 외부 제어의 기본 수단이며, 일부는 잡을 수 없어 강제 정책 도구로도 쓰인다.

## 왜 알아야 하나
- monitor.sh의 health check는 프로세스 식별·상태 분류에 의존
- B1-2 트러블슈팅(OOM·CPU·deadlock)의 진단은 프로세스 상태 해석
- 데몬 운영, 서비스 재시작, graceful shutdown 모두 시그널 기반
- 컨테이너의 PID 1 문제, init system의 reaping 등 모두 이 모델의 응용

## 프로세스 = (PID, address space, FDs, credentials, ...)

```
struct task_struct (커널 내부)
  ├─ pid, ppid, pgid, sid     식별·세션
  ├─ uid/gid + supplementary  신원
  ├─ mm_struct                메모리 매핑 (텍스트, 데이터, 힙, 스택, mmap된 파일)
  ├─ files_struct             열린 파일 디스크립터 테이블
  ├─ signal_struct            시그널 핸들러·마스크·pending
  ├─ state                    R/S/D/Z/T
  └─ ... (수십 개 더)
```

PID 1은 init/systemd — 모든 고아 프로세스를 입양해 reap. PID 0은 커널의 idle task (보이지 않음).

## fork-exec 모델

```
parent
  ├─ fork()  → 자식 PID 반환 (자식에게는 0 반환)
  │           ├─ 메모리 copy-on-write (성능 최적화 — write 시점에야 복사)
  │           ├─ 파일 디스크립터 복제 (close-on-exec 플래그 주의)
  │           └─ signal 핸들러는 그대로, pending 시그널은 비움
  │
  child (zero return from fork)
       └─ execve("/usr/bin/python", argv, envp)
                ├─ 가상 주소공간 새로 만듦
                ├─ 환경 변수는 envp[]로 전달
                └─ PID는 유지, 코드만 교체
```

★ Linux는 `fork()` 외에 `clone()`이 진짜 시스템 콜. 스레드(`pthread_create`)도 내부적으로 `clone()` + 공유 플래그 (CLONE_VM, CLONE_FILES 등).

★ vfork, posix_spawn은 fork+exec 패턴의 최적화 변형.

## 프로세스 상태

| state | 의미 | 인터럽트 가능 |
|---|---|---|
| `R` | runnable (실행 중 또는 큐 대기) | — |
| `S` | sleeping (시그널로 깰 수 있음) | ✅ |
| `D` | uninterruptible sleep (보통 디스크 I/O) | ❌ ★ |
| `Z` | zombie (exit 했지만 부모가 wait() 안 함) | — |
| `T` | stopped (SIGSTOP/SIGTSTP) | ✅ (SIGCONT) |
| `I` | idle kernel thread | — |

★ `D` 상태는 SIGKILL로도 못 죽임 — 디스크가 응답할 때까지 대기. 시스템 hang의 주범 중 하나 (NFS 끊김, dying disk).

## 시그널: 비동기 알림

표준 시그널은 1-31번 + realtime 32-64번. 주요:

| # | 이름 | 기본 동작 | 잡기 | 메모 |
|---|---|---|---|---|
| 1 | SIGHUP | Term | ✅ | 터미널 종료 / 데몬 reload 관습 |
| 2 | SIGINT | Term | ✅ | Ctrl+C |
| 3 | SIGQUIT | Core | ✅ | Ctrl+\, 코어 덤프 생성 |
| 9 | **SIGKILL** | Term | ❌ | 잡거나 무시 못함, 즉시 종료 |
| 11 | SIGSEGV | Core | ✅ | 잘못된 메모리 접근 |
| 13 | SIGPIPE | Term | ✅ | 닫힌 파이프에 write |
| 15 | **SIGTERM** | Term | ✅ | 정중한 종료 요청 (default) |
| 17 | SIGCHLD | Ign | ✅ | 자식 상태 변화 (wait()으로 reap) |
| 19 | SIGSTOP | Stop | ❌ | 프로세스 일시정지, 잡을 수 없음 |
| 18 | SIGCONT | Cont | ✅ | 정지된 프로세스 재개 |

기본 동작 변경: `signal()` (레거시), `sigaction()` (권장 — POSIX semantics 명시).

### SIGTERM vs SIGKILL — graceful shutdown의 핵심

```
SIGTERM (15)
  → 핸들러로 잡을 수 있음
  → cleanup 코드 실행 (FD close, lock 해제, state flush)
  → 프로세스가 명시적 exit() 또는 default Term

SIGKILL (9)
  → 커널이 직접 처리, 핸들러 우회
  → 메모리·FD는 OS가 회수하지만 application-level state 손상 가능
  → "최후의 수단"
```

운영 패턴: SIGTERM 후 grace period (보통 30s) → 안 끝나면 SIGKILL. systemd의 `KillMode`/`TimeoutStopSec`, K8s의 `terminationGracePeriodSeconds`가 이 모델.

### 좀비와 reaping

```
1. 자식 프로세스 exit() → 자원 대부분 해제
2. 종료 status는 보존 (struct task_struct에)
3. 부모가 wait() / waitpid() / SIGCHLD 핸들러로 회수
4. 회수 후 task_struct 완전 해제
```

부모가 회수 안 하면 → **좀비 (Z)** — PID 슬롯만 차지. PID 공간 고갈 위험 (`/proc/sys/kernel/pid_max`).

부모가 먼저 죽으면 → 자식은 PID 1(init)이 입양 → init은 항상 reap.

★ **컨테이너의 PID 1 문제**: 컨테이너 내 첫 프로세스가 init이 아닌 일반 앱이면, 자식 reap을 안 함 → 좀비 누적. `tini`, `dumb-init` 같은 minimal init 사용.

## 한 번 보자

```bash
ps -ef                       # System V 스타일
ps auxf                      # BSD + 프로세스 트리
ps -eLf                      # 스레드까지 (LWP 컬럼)
ps -p PID -o stat,wchan,cmd  # state + 무엇 대기 중인지

pgrep -lf agent_app          # 패턴 매치 + 명령줄
pidof bash
pstree -p                    # 프로세스 트리 시각화

# 시그널
kill PID                     # SIGTERM (default)
kill -TERM PID               # 동일
kill -9 PID                  # SIGKILL
kill -l                      # 시그널 이름·번호 매핑
killall -SIGUSR1 nginx       # 이름으로

# 상세 상태
cat /proc/PID/status         # state, uid/gid, threads, signals
cat /proc/PID/wchan          # 커널에서 무엇 대기 중인지
cat /proc/PID/maps           # 가상 메모리 매핑
ls -l /proc/PID/fd/          # 열린 파일 디스크립터들
cat /proc/PID/limits         # rlimit 값들

# 실시간
top                          # interactive
top -b -n 1 -p PID           # 한 번, 특정 PID만
htop                         # 더 사용자 친화 (별도 설치)
pidstat -p PID 1             # 1초 간격 통계 (sysstat 패키지)

# 트레이싱
strace -p PID                # syscall trace (live)
ltrace -p PID                # library call trace
```

## 흔한 함정

- **`ps`의 %CPU는 누적 평균** — 시작 후 (사용 시간 / 경과 시간). 순간 부하는 `top` 또는 `pidstat 1`.
- **PID 재사용** — exit한 PID는 곧 재사용됨. 오래된 PID 캐시 사용 시 다른 프로세스를 죽일 수 있음. `pgrep -f`로 명령줄 함께 검증.
- **`kill -9`로 인한 데이터 손상** — DB·파일 write 중간이면 일관성 깨짐. SIGTERM → wait → SIGKILL 패턴 준수.
- **D 상태 프로세스에 SIGKILL 안 통함** — 디스크 응답 대기. NFS·dying disk가 흔한 원인.
- **시그널 핸들러 안 안전하지 않은 함수 호출** — `printf`, `malloc` 등은 async-signal-safe X. `man 7 signal-safety` 참고.
- **fork bomb** — `:(){ :|:& };:` 같은 재귀 fork. cgroup의 `pids.max`로 방어.
- **wait() race** — `SIGCHLD` 핸들러에서 `waitpid(-1, ...)`로 모든 자식 reap (한 번에 여러 SIGCHLD가 합쳐질 수 있음).
- **`top`의 %CPU가 100% 초과** — 멀티코어에서 단일 프로세스가 N코어 점유 시 N×100%까지 가능 (Irix mode). `Shift+I`로 토글.
- **double fork 패턴** — 데몬화 시 부모-자식 관계를 init으로 옮기는 전통적 패턴. systemd/`Type=forking`이 이걸 처리.
- **SIGPIPE 무시** — 파이프라인 중 receiver가 죽으면 sender가 SIGPIPE로 종료. 네트워크 서버는 보통 SIGPIPE를 SIG_IGN으로 + EPIPE 직접 처리.

## B1-1 매핑

```bash
# monitor.sh — health check 스니펫
PID=$(pgrep -f 'agent_app\.py' | head -1)
if [ -z "$PID" ]; then
    log "[ERROR] agent_app.py not running"
    exit 1
fi

# 상태 검증
state=$(ps -o state= -p "$PID")
case "$state" in
    [RS])  log "[OK] PID=$PID state=$state" ;;
    D)     log "[WARN] PID=$PID in uninterruptible sleep" ;;
    Z)     log "[ERROR] PID=$PID is zombie" ; exit 1 ;;
    *)     log "[WARN] PID=$PID unexpected state=$state" ;;
esac

# 자원 측정 (% 누적 — 더 정확한 순간값은 /proc/PID/stat 두 시점 차이)
ps -p "$PID" -o %cpu=,%mem= --no-headers
```

cron이 monitor.sh 실행 = cron 데몬이 fork → exec /bin/sh → exec monitor.sh. 끝나면 cron이 wait()으로 reap → 좀비 안 생김.

## 인접 토픽

- **process group, session, controlling terminal** — `setpgid`, `setsid`, terminal 제어, job control
- **prctl(2)** — 프로세스 속성 세밀 제어 (PR_SET_NAME, PR_SET_PDEATHSIG, no_new_privs 등)
- **cgroups (v1/v2)** — 리소스 격리·제한·계측 (CPU, 메모리, IO, PID)
- **namespace** — PID/network/mount/user/UTS/IPC namespace로 컨테이너 격리
- **eBPF + tracing** — `bpftrace`, `perf`, `bcc`로 프로세스 동작 관측
- **systemd** — service unit으로 프로세스 lifecycle 관리 (`Type=`, `KillSignal=`, `Restart=`)
- **OOM killer** — 메모리 압박 시 OS가 프로세스 골라 SIGKILL (`/proc/PID/oom_score_adj`로 가중치)

## 참고
- `man 7 signal`, `man 7 signal-safety`
- `man 2 fork`, `man 2 execve`, `man 2 clone`, `man 2 wait`
- `man 5 proc` — `/proc/PID/*` 파일 의미 전체

---
출처: B1-1 (Layer 1.5) · 학습일: 2026-05-07
