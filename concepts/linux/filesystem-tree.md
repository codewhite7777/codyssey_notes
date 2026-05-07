# Linux 파일시스템 트리

## 한 줄 정의
모든 파일과 디렉토리가 **단일 루트(`/`)에서 시작하는 표준화된 트리 구조** — 어떤 리눅스 머신에서도 같은 종류의 파일은 같은 위치에서 찾을 수 있게 하는 약속(FHS = Filesystem Hierarchy Standard).

## 왜 필요한가

### 만약 표준이 없다면
- 머신마다 설정 파일이 다른 위치 → 자동화·운영 불가능
- "이 로그 파일 어디 있지?" → 매번 검색
- 패키지 매니저가 어디에 파일을 깔지 모름

### 표준이 있어서 가능한 것
- `/etc`에 가면 시스템 설정이 있다 → 어떤 머신이든 일관
- `/var/log`에 가면 로그가 있다 → 모니터링 도구 일반화 가능
- 운영 자동화 스크립트가 portable

## 핵심 원리

```
/                    ← 루트 (모든 게 여기서 시작)
├── etc/             설정 파일 (configuration)
├── var/             변경되는 데이터 (logs, caches, mail, …)
│   └── log/         로그 (이번 과제: /var/log/agent-app/)
├── home/            일반 사용자 홈 (예: /home/agent-admin/)
├── proc/            프로세스/커널 정보 (가상 파일시스템 — 디스크에 없음!)
├── tmp/             임시 (재부팅 시 자주 비워짐)
├── usr/             사용자 프로그램 (대부분의 설치된 명령)
│   ├── bin/         일반 사용자용 실행 파일
│   └── sbin/        관리자용 실행 파일
├── bin/             핵심 명령 (ls, cp, …) — usr/bin과 합쳐지는 추세
├── sbin/            핵심 관리 명령 (mount, …)
├── opt/             3rd-party 패키지 (수동 설치 등)
├── root/            root 사용자 홈 (일반 사용자와 분리)
└── dev/             장치 파일 (디스크, 터미널 등)
```

## 이번 과제(B1-1)에서 만나는 것

| 경로 | 무엇 | 왜 거기 |
|---|---|---|
| `/etc/ssh/sshd_config` | SSH 데몬 설정 | 시스템 설정은 `/etc` |
| `/home/agent-admin/agent-app/` | 앱 홈 (`AGENT_HOME`) | 사용자별 홈은 `/home` |
| `/home/agent-admin/agent-app/upload_files/` | 업로드 데이터 | 앱 데이터는 보통 앱 홈 아래 |
| `/var/log/agent-app/` | 모니터링 로그 | 변경되는 로그는 `/var/log` |
| `/proc/<PID>/` | 프로세스 정보 (CPU/메모리 측정 시) | 프로세스 정보는 가상 FS `/proc` |

## 흔한 오해

- ❌ "`/proc/cpuinfo`를 디스크에서 찾을 수 있을 것" → ✅ `/proc`는 **가상 파일시스템**. 디스크에 없음. 커널이 즉석에서 만들어 보여주는 것.
- ❌ "로그는 항상 `/var/log` 아래" → ✅ 대부분 그렇지만 도커·앱마다 자기 위치에 둘 수 있음. 그래도 **표준은 `/var/log`**.
- ❌ "사용자 데이터는 `/home`에만" → ✅ 시스템 사용자(예: `nginx`)는 종종 `/var/lib/nginx` 같은 곳을 홈으로 씀.
- ❌ "`/tmp`는 안전하게 영구 저장 가능" → ✅ 재부팅 시 비워질 수 있음. 영구 데이터는 `/var/`나 홈에.

## 자주 쓰는 명령

```bash
ls /                    # 루트 트리 한눈에
ls -la /etc/ | head     # 설정 파일들 일부
df -h /                 # 루트 파티션 사용량
du -sh /var/log/        # 특정 디렉토리 용량
tree -L 2 /var          # /var 2단계 깊이까지 (tree 미설치면 'find /var -maxdepth 2')
file /etc/passwd        # 파일 종류 확인
stat /etc/ssh/sshd_config   # 메타데이터 (권한, 수정시간 등)
```

## 관련 개념
- [users-and-groups.md](./users-and-groups.md) — 누가 어디에 접근할 수 있는지
- [file-permissions.md](./file-permissions.md) — `/etc` 같은 곳을 보호하는 메커니즘
- [process-and-signals.md](./process-and-signals.md) — `/proc`로 프로세스 관측

## 출처
- B1-1 시스템 관제 자동화 스크립트 (Layer 1.1)
- 학습일: 2026-05-07
- 참고: `man hier` (리눅스 파일시스템 계층 매뉴얼), [Filesystem Hierarchy Standard](https://refspecs.linuxfoundation.org/fhs.shtml)
