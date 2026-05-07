# 셸 환경 = 출근하는 직원의 출근 메모

## 한 마디로
셸(우리 명령을 받아 실행해주는 직원)이 출근할 때 **어떤 메모를 읽고 어떤 환경 정보를 들고** 시작하는지 결정하는 시스템.

## 왜 알아야 해?

- 매번 SSH 접속할 때마다 환경 변수를 손으로 안 쳐도 자동
- "왜 SSH 들어가면 환경변수가 안 먹지?" 같은 함정 피함
- cron이 시간 맞춰 자동 실행할 때 "왜 명령을 못 찾지?" 같은 문제 해결

## 핵심 비유: 셸 = 직원

```
사용자 (당신)
    │ 명령 입력
    ▼
직원(셸) ─── 명령을 OS에 전달 ──► 컴퓨터
    │
    │ 출근 시 메모 읽음
    ▼
.bash_profile / .bashrc (출근 메모)
```

## 직원에게는 두 종류의 출근이 있음

### 1. 정식 출근 (Login)
- SSH 접속, `su -` 등
- 읽는 메모: **`.bash_profile`** (없으면 `.profile`)
- "오늘 어떤 환경에서 일할지" 처음 설정

### 2. 일상 출근 (Non-login)
- 새 터미널 탭 열기, `bash` 명령
- 읽는 메모: **`.bashrc`**
- "이미 출근한 상태에서 보조 직원 추가"

### 3. 메모 안 읽는 출근 (스크립트, cron) ★
- 셸 스크립트 실행, cron 자동 실행
- **메모 안 읽음**
- "잠깐 왔다가 일만 하고 가는 임시 직원"
- → 환경 변수 거의 비어있음 (cron의 큰 함정)

## 그림으로 보기

```
             어떤 출근?
                │
        ┌───────┼───────┐
        ▼       ▼       ▼
       정식    일상    스크립트/
       출근    출근    cron
        │       │       │
        ▼       ▼       ▼
   .bash_     .bashrc   ❌ 안 읽음
   profile    만 읽음   (PATH도 거의 비어있음)
   읽음
```

## 환경 변수 = 직원이 기억하는 정보

```bash
MY_VAR=hello              # 직원만 기억 (다른 직원에게 안 알려줌)
export MY_VAR             # "부하 직원에게도 알려줘" 표시
```

→ `export` 안 하면 **자식 프로세스(부하 직원)는 그 변수 못 봄**.

확인:
```bash
MY_VAR=hello
echo $MY_VAR              # hello (현재 셸은 봄)
bash -c 'echo $MY_VAR'    # 빈 줄 (자식은 못 봄)

export MY_VAR
bash -c 'echo $MY_VAR'    # hello (이제 봄)
```

## 일반적인 패턴

대부분 `.bash_profile`에 이렇게 둠:
```bash
# .bash_profile (정식 출근 메모)

# 일상 출근 메모도 같이 읽기
if [ -f ~/.bashrc ]; then
    source ~/.bashrc
fi

# 정식 출근 시에만 할 일
export PATH=$PATH:$HOME/bin
```

→ SSH로 들어가도, 새 터미널 열어도 같은 환경.

## 한 번 보자

```bash
echo $SHELL           # 어느 직원(셸)인지 (예: /bin/bash)
env                   # 직원이 기억하는 정보 전체
env | grep AGENT      # AGENT_ 시작 변수만
echo $HOME            # 본인 집 경로
echo $PATH            # 명령 검색 경로

# .bashrc 다시 읽기 (재출근 안 하고)
source ~/.bashrc
. ~/.bashrc           # source의 줄임 (똑같음)
```

## 자주 헷갈리는 것

- **SSH 들어가면 `.bashrc`가 읽힘** → ❌ SSH는 정식 출근 → `.bash_profile`. `.bashrc`는 명시적 source 해야.
- **`export` 없이 변수 정의해도 자식 스크립트에서 보임** → ❌ **export 필수**.
- **cron에서 환경 변수가 없는 건 셸 버그** → ❌ cron은 메모 안 읽는 임시 직원. 정상 동작.
- **세 가지 메모 다 만들어야** → ❌ 셸은 우선순위로 한 개만 읽음.

## 이번 과제(B1-1)에서

`AGENT_HOME` 등을 SSH 접속 시 자동 설정:

```bash
# /home/agent-admin/.bash_profile 에:
export AGENT_HOME=/home/agent-admin/agent-app
export AGENT_PORT=15034
export AGENT_UPLOAD_DIR=$AGENT_HOME/upload_files
export AGENT_KEY_PATH=$AGENT_HOME/api_keys/t_secret.key
export AGENT_LOG_DIR=/var/log/agent-app

# .bashrc도 같이 읽기
[ -f ~/.bashrc ] && source ~/.bashrc
```

★ **cron이 monitor.sh 실행할 때는 이 환경 못 받음** — 스크립트 안에서 직접 export하거나 첫 줄에서 source 해야. (Layer 5에서 자세히)

## 기술 용어 풀이

- **셸(shell)** — 명령어를 받아 OS에 전달하는 직원. bash, zsh, sh 등.
- **환경 변수(environment variable)** — 직원이 기억하는 키=값 정보.
- **export** — 자식 프로세스에 변수 전달하라는 표시.
- **source / `.`** — 메모(파일)를 다시 읽어서 환경 갱신.
- **login shell** — 정식 출근 시작 셸.
- **non-login shell** — 일상 출근 셸.
- **interactive shell** — 사람이 키보드로 명령 입력하는 셸.

## 더 알아보고 싶으면
- 명령: `man bash` (INVOCATION 섹션), `man env`
- 다음 노트: [process-and-signals.md](./process-and-signals.md) — 직원이 실제로 일하는 단위(프로세스)

## 출처
- B1-1 (Layer 1.4)
- 학습일: 2026-05-07
