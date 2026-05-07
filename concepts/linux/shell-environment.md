# 셸 환경과 초기화

## 개요
Bash는 시작 모드(login vs interactive)에 따라 다른 startup 파일을 source하며, 이를 통해 환경 변수·alias·함수·셸 옵션을 설정한다. 환경 변수는 `execve()` 호출 시 자식 프로세스에 복사되어 전파되므로, 어떤 시점에 어떻게 설정되는지가 모든 자동화의 기반이 된다.

## 왜 알아야 하나
- SSH·sudo·cron은 각각 다른 시작 모드 → 환경이 달라짐
- "왜 cron에서 명령을 못 찾지" 같은 함정의 근본 원인
- 컨테이너 ENTRYPOINT, systemd 유닛의 `Environment=` 설정도 동일 모델
- 환경 변수 leak이 보안 취약점이 되는 경우 (LD_PRELOAD, PATH injection)

## 셸의 두 차원

```
                    Interactive       Non-interactive
                  ┌────────────────┬─────────────────┐
       Login      │  사용자 SSH     │  드뭄 (`bash -l`)│
                  │  `su -`         │                 │
                  ├────────────────┼─────────────────┤
       Non-login  │  새 터미널 탭    │  스크립트 실행   │
                  │  `bash`         │  cron job        │
                  └────────────────┴─────────────────┘
```

판별:
- `[[ $- == *i* ]]` → interactive (`-i` 플래그 set)
- `shopt -q login_shell` → login

## Startup 파일 로딩 순서

### Login + Interactive (예: SSH)
```
1. /etc/profile
   └─ 보통 /etc/profile.d/*.sh 모두 source
2. ~/.bash_profile     (있으면)
   ├─ 또는 ~/.bash_login (없으면)
   └─ 또는 ~/.profile  (둘 다 없으면)
   ★ 보통 .bashrc를 source하도록 작성
```

### Non-login + Interactive (예: 새 터미널)
```
1. /etc/bash.bashrc
2. ~/.bashrc
```

### Non-interactive (스크립트, cron)
```
- 어느 startup 파일도 자동 source하지 않음
- 단 BASH_ENV 환경변수가 set되어 있으면 그 파일을 source
- $0이 'sh'로 호출되면 ENV 변수의 파일
```

→ **cron은 거의 빈 환경**: 보통 `PATH=/usr/bin:/bin`, `SHELL=/bin/sh`, `HOME=$HOME`. `.bashrc` 안 읽음.

## 환경 변수 vs 셸 변수

```bash
foo=hello                # 셸 변수 (현재 프로세스에만)
export foo               # environment list에 추가 → 자식 프로세스 상속
export bar=world         # 정의 + export 동시
declare -x baz=value     # export와 동일

unset foo                # 제거
```

**내부적으로**:
- 셸 변수: 셸 프로세스의 메모리 데이터 구조 (key-value)
- 환경 변수: `execve(path, argv, envp)`의 `envp[]` 배열로 자식에게 복사

`/proc/PID/environ` 으로 임의 프로세스의 환경 확인 가능 (NULL 구분, 권한 있으면).

## 일반적 패턴

```bash
# ~/.bash_profile  (login shell — SSH 진입점)
[ -f ~/.bashrc ] && . ~/.bashrc        # interactive 설정도 같이
export PATH="$HOME/.local/bin:$PATH"   # login에서만 PATH 추가

# ~/.bashrc  (interactive shell — alias·prompt 등)
[[ $- == *i* ]] || return              # non-interactive면 즉시 종료
alias ll='ls -alF'
PS1='\u@\h:\w\$ '
```

★ `.bashrc` 첫 줄의 `[[ $- == *i* ]] || return` 중요 — `.bashrc`가 non-interactive에서 source되면 출력 명령이 SSH 핸드셰이크를 깨거나 stdin/stdout을 오염시킴 (특히 `scp`, `sftp` 깨짐).

## 한 번 보자

```bash
echo $0                      # 셸 호출 방식 (예: -bash → login)
shopt login_shell            # off/on
echo $-                      # 활성 옵션 (i가 있으면 interactive)

env                          # environment list 전체
env | grep -i agent          # 필터
declare -p VARNAME           # 변수 메타데이터 (export 여부 등)
declare -px                  # exported 변수만

# 자식 프로세스 환경 확인
bash -c 'echo $foo'
sudo -i env                  # sudo가 환경 어떻게 처리하는지

# startup 추적
bash -x -l                   # login shell trace 모드
bash --rcfile /tmp/myrc      # 다른 rc로 시작

# 임의 프로세스 환경 (권한 있으면)
xargs -0 -L1 < /proc/$$/environ
```

## 흔한 함정

- **SSH 진입 후 `.bashrc` 안 읽힘** — login shell이라 `.bash_profile`만. `.bash_profile`에서 `.bashrc`를 source하지 않으면 alias·함수 다 누락.
- **non-interactive에서 alias 안 먹음** — Bash 기본적으로 non-interactive 셸에서 alias expansion off (`shopt expand_aliases` 필요).
- **cron의 PATH 부재** — `/usr/local/bin`, `~/.local/bin` 등 모두 누락. 절대 경로 사용 또는 crontab 상단에 `PATH=` 명시.
- **`sudo`의 환경 reset** — sudoers `env_reset` 기본 정책으로 대부분 환경 변수 drop. `--preserve-env` 또는 sudoers `env_keep` 명시 필요.
- **`export VAR=$(cmd)` 의 함정** — `cmd`가 실패해도 export는 성공 (exit code는 마지막 명령인 `export`의 것 = 0). `set -e`와 조합 시 검출 안 됨. 분리해서: `VAR=$(cmd) || exit; export VAR`.
- **environ size 한계** — `MAX_ARG_STRLEN` (보통 128KB) 또는 `ARG_MAX` (보통 2MB) 초과 시 `execve` 실패 (`E2BIG`). 환경에 거대한 데이터 넣으면 깨짐.
- **PATH 보안** — sudo로 실행되는 스크립트가 상대 경로를 쓰면 PATH injection 공격 가능. secure_path 사용.
- **배포판별 default startup** — Debian의 `/etc/skel/.bashrc`, RHEL의 `/etc/profile.d/`, macOS Bash 3.x의 다른 동작.

## B1-1 매핑

```bash
# /home/agent-admin/.bash_profile
export AGENT_HOME=/home/agent-admin/agent-app
export AGENT_PORT=15034
export AGENT_UPLOAD_DIR=$AGENT_HOME/upload_files
export AGENT_KEY_PATH=$AGENT_HOME/api_keys/t_secret.key
export AGENT_LOG_DIR=/var/log/agent-app

# .bashrc도 같이 (interactive 환경 일관성)
[ -f ~/.bashrc ] && . ~/.bashrc
```

★ **cron 실행 monitor.sh의 환경 함정**:
1. PATH 없음 → 절대 경로 (`/usr/bin/ps`) 사용 또는 스크립트 첫 줄에서 `PATH=...` 설정
2. `AGENT_*` 환경변수 없음 → 스크립트 안에서 source 또는 export
3. `LANG`/`LC_ALL` 없음 → `ps`/`date` 출력 형식이 달라질 수 있음 → `LC_ALL=C` 명시 권장 (POSIX 일관성)

```bash
# monitor.sh 상단 권장 패턴
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
export LC_ALL=C
[ -f /home/agent-admin/.bash_profile ] && . /home/agent-admin/.bash_profile
```

## 인접 토픽

- **systemd unit `Environment=` / `EnvironmentFile=`** — 서비스 환경 관리의 표준
- **`/etc/environment`** — PAM이 모든 로그인에 set (셸 무관, KEY=VALUE 단순 형식)
- **dotenv 패턴** — 앱별 `.env` 파일 (보안 주의 — 권한 관리)
- **direnv / mise / asdf** — 디렉토리별 자동 환경 전환
- **secrets management** — 환경 변수의 secret 저장은 안티패턴 (HashiCorp Vault, AWS Secrets Manager, 1Password CLI)
- **shellcheck** — 셸 스크립트 정적 분석 (필수 도구)

## 참고
- `man bash` — INVOCATION, FILES 섹션
- `man 7 environ`
- `man 5 crontab` — cron 환경 모델
- `man 8 sudoers` — env_reset, env_keep 정책

---
출처: B1-1 (Layer 1.4) · 학습일: 2026-05-07
