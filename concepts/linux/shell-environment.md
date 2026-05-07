# 셸 환경과 초기화 파일

## 한 줄 정의
셸이 시작될 때 **어떤 init 파일을 읽는지**는 시작 종류(login vs interactive)에 따라 다르며, 이를 통해 환경 변수와 별칭이 적절한 시점에 설정됨.

## 왜 필요한가
- 새 터미널/SSH 접속할 때마다 환경 변수 자동 적용
- `AGENT_HOME` 같은 앱 환경 변수를 매번 손으로 안 쳐도 되게
- 셸별로 다른 초기 설정 (로그인 시에는 `motd` 출력, 일반 셸에는 별칭만)

→ 안 알면 "왜 SSH로 들어가면 환경변수가 안 먹지?" 같은 함정에 빠짐

## 핵심 원리

### 셸 시작의 두 축

```
                    Interactive       Non-interactive
                    (사용자가 침)       (스크립트 등)
                  ┌───────────────┬─────────────────┐
       Login      │      A        │   (드뭄)         │
   (SSH 등)        │               │                 │
                  ├───────────────┼─────────────────┤
       Non-login  │      B        │      C          │
   (새 터미널 등)   │               │ (cron, 스크립트) │
                  └───────────────┴─────────────────┘
```

### 각 케이스에서 읽는 파일 (bash 기준)

**A. Login + Interactive** (SSH 접속, `su -`)
- `/etc/profile` → 시스템 전역
- `~/.bash_profile` → 있으면 이거만
- 없으면 `~/.bash_login` → 그것도 없으면 `~/.profile`
- (`~/.bashrc`는 **자동으로 안 읽음** — 보통 `.bash_profile`에서 source)

**B. Non-login + Interactive** (새 터미널 탭, `bash` 명령)
- `/etc/bash.bashrc`
- `~/.bashrc`

**C. Non-interactive** (셸 스크립트 실행, cron job)
- 둘 다 안 읽음 (기본적으로)
- 환경 변수 == 부모 프로세스에서 상속받은 것뿐

> ★ **cron의 함정**: cron은 C 케이스. PATH도 거의 비어있고 (`/usr/bin:/bin` 정도) `~/.bashrc` 안 읽음. → cron job에서 환경 변수 필요하면 명시적으로 export하거나 `BASH_ENV` 설정.

### 일반적인 패턴 (관습)

대부분의 배포판은 이렇게 둠:
```bash
# ~/.bash_profile
if [ -f ~/.bashrc ]; then
    source ~/.bashrc          # login shell도 .bashrc 읽도록
fi
# 그 후 login-only 설정 (PATH 추가 등)
export PATH=$PATH:$HOME/bin
```

→ SSH(login)나 새 터미널(non-login)이나 같은 환경.

### export의 의미
```bash
MY_VAR=hello              # 현재 셸에만 존재
export MY_VAR             # 이 셸이 만든 자식 프로세스에도 전파
export AGENT_HOME=/opt/x  # 정의 + 전파 동시에
```

→ `export` 안 하면 `python script.py`에서 그 변수 못 봄.

### 환경 변수 vs 셸 변수
```bash
MY_VAR=hello       # 셸 변수 (현재 셸 내부에만)
echo $MY_VAR       # hello (현재 셸에서는 보임)
bash -c 'echo $MY_VAR'   # 빈 줄 (자식 프로세스는 못 봄)

export MY_VAR
bash -c 'echo $MY_VAR'   # hello (이제 자식도 봄)
```

## 자주 쓰는 명령

```bash
echo $SHELL                  # 로그인 셸 종류 (예: /bin/bash)
echo $0                      # 현재 셸 시작 방식 (-bash면 login)
env                          # 현재 환경 변수 전체
env | grep AGENT             # AGENT_ 시작 변수들만
printenv AGENT_HOME          # 특정 변수만
source ~/.bashrc             # init 파일 다시 읽기 (재로그인 없이)
. ~/.bashrc                  # source의 줄임 (POSIX 호환)
bash --login                 # login shell로 새로 시작
echo $$                      # 현재 셸의 PID
```

## 흔한 오해

- ❌ ".bashrc는 SSH 접속 시 자동 읽힘" → ✅ SSH는 보통 login → `.bash_profile` 읽음. `.bashrc`는 명시적 source 없으면 무시.
- ❌ "export 없이 변수 정의해도 자식 스크립트에서 보임" → ✅ 안 보임. **export가 필수**.
- ❌ "cron job에서 환경 변수가 안 먹어 — 셸 버그" → ✅ 버그가 아니라 cron이 non-interactive non-login. 환경 거의 비어있음.
- ❌ ".profile / .bash_profile / .bash_login 다 만들어야" → ✅ 셸은 셋 중 **첫 번째 발견된 것 하나만** 읽음.
- ❌ "환경 변수 변경하면 다른 터미널에도 반영" → ✅ 셸별로 독립. 새로 띄우는 셸만 init 파일을 다시 읽어 적용.

## 이번 과제(B1-1)에서

```bash
# AGENT_HOME 등을 SSH 접속 시 자동 설정하려면:

# /home/agent-admin/.bash_profile
export AGENT_HOME=/home/agent-admin/agent-app
export AGENT_PORT=15034
export AGENT_UPLOAD_DIR=$AGENT_HOME/upload_files
export AGENT_KEY_PATH=$AGENT_HOME/api_keys/t_secret.key
export AGENT_LOG_DIR=/var/log/agent-app

# .bashrc도 같이 읽고 싶으면 (대부분 그렇게 함):
[ -f ~/.bashrc ] && source ~/.bashrc
```

> ★ cron으로 실행되는 monitor.sh는 이 환경을 못 받음. 해결책:
> 1. 스크립트 안에서 직접 export
> 2. crontab 줄에서 `MAILTO=...` 위에 환경 변수 정의
> 3. 스크립트 첫머리에서 `source /home/agent-admin/.bash_profile`

이 함정은 Layer 5(cron)에서 깊이 다룸.

## 관련 개념
- [filesystem-tree.md](./filesystem-tree.md) — 홈 디렉토리 위치
- [process-and-signals.md](./process-and-signals.md) — 환경 변수는 프로세스 fork 시 자식에게 복사됨
- (예정) cron-environment.md — Layer 5

## 출처
- B1-1 (Layer 1.4)
- 학습일: 2026-05-07
- 참고: `man bash` (INVOCATION 섹션), `man env`
