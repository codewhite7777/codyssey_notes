# 셸 환경과 초기화

> **TLDR** · 셸 시작은 `(login × interactive)` 4-cell 매트릭스. SSH는 login → `.bash_profile`, 새 터미널은 non-login → `.bashrc`, cron은 non-interactive → **어느 init도 안 읽음** (PATH 거의 비어있음). 이게 cron 함정의 근본 원인.

## 개요

셸은 사용자가 OS와 대화하는 가장 기본적인 인터페이스다. 우리가 키보드로 친 명령은 셸이 받아서 해석하고 자식 프로세스로 fork·exec해 실행한다. 그런데 셸은 단순한 명령 실행기가 아니라, 자기만의 환경(환경 변수·alias·함수·옵션)을 들고 다니는 프로세스이며, 그 환경이 어떻게 초기화되는지가 모든 자동화의 기반이 된다.

Bash는 시작 모드(login vs interactive)에 따라 다른 startup 파일을 source하며, 각 모드는 historical하게도 의미가 다르다. 환경 변수는 `execve()` 호출 시 자식 프로세스에 복사되어 전파되므로, 어떤 시점에 어떤 변수가 어떻게 설정되는지가 자식 프로세스의 동작을 결정한다. 이 초기화 메커니즘을 이해하지 못하면 "왜 SSH로 들어가면 환경변수가 안 먹히지", "왜 cron 작업이 명령을 못 찾지" 같은 함정에 자주 빠지게 된다.

## 왜 알아야 하나

셸 환경 모델은 표면적으로는 단순한 점 파일(`.bashrc`, `.bash_profile`) 관리처럼 보이지만, 실제로는 시스템 자동화 전반의 기반이다. SSH·sudo·cron은 각각 다른 시작 모드로 셸을 띄우기 때문에 같은 사용자의 같은 머신에서도 환경이 다르다. 이를 모르면 "로컬에서는 잘 되는데 cron으로 돌리면 안 됨" 같은 디버깅하기 어려운 문제가 반복된다.

운영 측면에서 환경 변수의 leak은 보안 취약점이 되기도 한다. `LD_PRELOAD`를 통한 라이브러리 주입, PATH injection을 통한 트로이목마 명령 실행 등이 모두 환경 모델의 약점을 활용한다. 컨테이너 시대에 와서는 systemd unit의 `Environment=`, Docker의 ENTRYPOINT 환경, Kubernetes의 env 주입 등이 모두 같은 모델 위에서 작동하므로, 셸 환경의 정확한 이해는 인프라 전반에 걸쳐 적용된다.

이번 과제도 이 모델의 직접적 영향을 받는다. `AGENT_HOME` 같은 환경 변수를 SSH 접속 시 자동 설정하려면 어느 파일에 두어야 하는지, cron이 monitor.sh를 실행할 때 그 환경을 받지 못하는 이유는 무엇인지 — 이런 질문들이 모두 셸 초기화 모델로 답이 나온다.

## 셸의 두 차원

Bash 셸의 시작 동작은 두 차원의 조합으로 결정된다 — **login 여부**와 **interactive 여부**다. 이 두 축의 조합으로 4가지 시나리오가 만들어진다.

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

**Login shell**은 새로운 사용자 세션의 시작점에 해당한다. SSH 접속, `su -` (대시 핵심), 콘솔 로그인 등이 login shell을 띄운다. 역사적으로 login shell은 "이 사용자가 처음 로그인했으니 환경을 처음부터 설정하자"는 의도를 가진다.

**Interactive shell**은 사용자와 대화하는 셸이다. 키보드로 명령을 입력받고 결과를 보여준다. 새 터미널을 열거나 `bash` 명령을 그냥 실행하면 interactive non-login shell이 시작된다.

**Non-interactive shell**은 스크립트나 cron 작업처럼 사람이 직접 명령을 치지 않는 경우다. 이 경우 셸은 "한 번 일하고 끝낼" 임시 프로세스에 가깝다.

판별은 다음과 같이 한다 — `[[ $- == *i* ]]`가 참이면 interactive (`-i` 플래그가 set), `shopt -q login_shell`이 참이면 login.

## Startup 파일 로딩 순서

각 시나리오마다 Bash가 source하는 파일이 다르다. 이 차이가 자동화의 함정의 절반을 만든다.

```mermaid
flowchart TD
    A[셸 시작] --> B{login shell?}
    B -->|Yes<br/>SSH·su -| C[/etc/profile<br/>+ /etc/profile.d/*]
    C --> D{~/.bash_profile<br/>있음?}
    D -->|있음| E[.bash_profile 읽음<br/>★ 보통 여기서 .bashrc도 source]
    D -->|없음| F[.bash_login 또는<br/>.profile 읽음]

    B -->|No| G{interactive?}
    G -->|Yes<br/>새 터미널| H[/etc/bash.bashrc<br/>+ ~/.bashrc]
    G -->|No<br/>스크립트·cron| I[★ 어느 init도 안 읽음<br/>PATH 거의 비어있음<br/>= cron 함정의 원인]

    style I fill:#ffcccc
    style E fill:#cce5ff
    style F fill:#cce5ff
    style H fill:#cce5ff
```

**Login + Interactive (예: SSH)** 의 경우, Bash는 먼저 `/etc/profile`을 읽고(보통 `/etc/profile.d/*.sh`를 모두 source한다), 그 다음 사용자 홈에서 `~/.bash_profile`을 찾는다. 만약 없으면 `~/.bash_login`을, 그것도 없으면 `~/.profile`을 읽는다. 셋 중 첫 번째 발견된 것 하나만 읽고 나머지는 무시된다. 중요한 점은 **`~/.bashrc`는 자동으로 안 읽힌다**는 사실이다 — 보통 관습적으로 `.bash_profile`에서 `.bashrc`를 source하도록 작성한다.

**Non-login + Interactive (예: 새 터미널 탭)** 의 경우는 더 단순하다. `/etc/bash.bashrc`와 `~/.bashrc`만 읽힌다. login startup 파일은 무시된다.

**Non-interactive (스크립트, cron)** 의 경우는 거의 비어 있다. 어느 startup 파일도 자동 source하지 않는다. 단 `BASH_ENV` 환경변수가 set되어 있으면 그 파일을 source하고, `$0`이 `sh`로 호출되면 `ENV` 변수의 파일을 읽는다. 그래서 **cron은 거의 빈 환경**으로 시작한다 — 보통 `PATH=/usr/bin:/bin`, `SHELL=/bin/sh`, `HOME=$HOME` 정도만 set되어 있고 `~/.bashrc`는 안 읽힌다. 이게 cron 함정의 원인이다.

## 환경 변수 vs 셸 변수

환경의 또 다른 차원은 변수의 종류다. 단순히 변수를 정의하는 것과 자식 프로세스에 전파되는 환경 변수로 만드는 것은 다른 동작이다.

```bash
foo=hello                # 셸 변수 (현재 프로세스에만)
export foo               # environment list에 추가 → 자식 프로세스 상속
export bar=world         # 정의 + export 동시
declare -x baz=value     # export와 동일 (더 명시적)
```

내부적으로는 다음과 같다. 셸 변수는 셸 프로세스의 메모리 안에 있는 key-value 데이터 구조다. 환경 변수는 `execve(path, argv, envp)` 시스템 콜의 `envp[]` 배열로 자식 프로세스에 전달된다. 자식 프로세스는 이 배열을 자기 메모리에 복사해서 사용한다.

이 모델의 직접적 결과는 **`export` 없이 정의된 변수는 자식 프로세스에서 안 보인다**는 것이다.

```bash
foo=hello
echo $foo                # hello (현재 셸은 봄)
bash -c 'echo $foo'      # 빈 줄 (자식은 못 봄)

export foo
bash -c 'echo $foo'      # hello (이제 봄)
```

이 사실은 모든 자동화의 기본기다. 스크립트가 외부 명령에 환경 변수를 전달하려면 export가 필요하고, 환경 변수의 값을 보고 싶으면 `/proc/PID/environ`을 (권한 있다면) 직접 읽을 수 있다.

## 일반적 패턴: .bash_profile에서 .bashrc 부르기

대부분의 배포판은 login shell과 non-login shell의 환경을 일관되게 만들기 위해 다음 패턴을 사용한다.

```bash
# ~/.bash_profile  (login shell — SSH 진입점)
[ -f ~/.bashrc ] && . ~/.bashrc        # interactive 설정도 같이
export PATH="$HOME/.local/bin:$PATH"   # login에서만 PATH 추가

# ~/.bashrc  (interactive shell — alias·prompt 등)
[[ $- == *i* ]] || return              # non-interactive면 즉시 종료
alias ll='ls -alF'
PS1='\u@\h:\w\$ '
```

`.bash_profile`이 `.bashrc`를 source하는 이유는, login shell로 들어와도 alias나 prompt 같은 interactive 설정이 적용되도록 하기 위해서다. 이 한 줄이 없으면 SSH로 들어와서는 `ll` 같은 alias가 작동하지 않는데 새 터미널을 열어서는 작동하는 이상한 비대칭이 생긴다.

`.bashrc`의 첫 줄 `[[ $- == *i* ]] || return`은 매우 중요하다. 이 가드가 없으면 `.bashrc`가 non-interactive 환경에서도 source되어 출력 명령이 SSH 핸드셰이크를 깨거나 stdin/stdout을 오염시킬 수 있다. 특히 `scp`나 `sftp`가 이런 이유로 깨지는 경우가 흔하다 — 이 도구들은 SSH 채널 위에서 protocol을 주고받는데, `.bashrc`가 무언가를 출력하면 protocol이 corrupted된다.

## 한 번 보자

환경 모델을 직접 확인하는 명령들이다. 먼저 현재 셸의 정체를 파악한다.

```bash
echo $0                      # 셸 호출 방식 (예: -bash → login)
shopt login_shell            # off/on
echo $-                      # 활성 옵션 (i가 있으면 interactive)
```

`echo $0`이 `-bash`처럼 대시로 시작하면 login shell이고, 그냥 `bash`면 non-login이다. `shopt login_shell`로도 같은 정보를 얻는다. 이 차이를 직접 확인해보면 모델이 머리에 박힌다.

이어서 환경 변수를 살펴본다.

```bash
env                          # environment list 전체
env | grep -i agent          # AGENT_ 시작 변수만
declare -p VARNAME           # 변수 메타데이터 (export 여부 등)
declare -px                  # exported 변수만
```

`env`는 현재 프로세스의 환경 변수를 출력한다. `declare -p`는 export 여부까지 보여줘서, 셸 변수와 환경 변수의 차이를 확인할 때 유용하다.

자식 프로세스에 환경이 어떻게 전달되는지 보려면 직접 비교하면 된다.

```bash
foo=hello                    # 셸 변수만 정의
bash -c 'echo "child sees: $foo"'   # 빈 줄

export foo                   # export
bash -c 'echo "child sees: $foo"'   # hello

sudo -i env                  # sudo가 환경 어떻게 처리하는지 (기본 reset)
sudo -E -i env               # --preserve-env 했을 때
```

`sudo`는 기본적으로 환경을 대부분 reset하는데, `--preserve-env`로 보존할 수 있다. 이 동작은 sudoers 정책의 `env_reset`/`env_keep`/`env_check`로 세밀하게 조절된다.

마지막으로 임의 프로세스의 환경을 볼 수 있다 (권한이 있다면).

```bash
xargs -0 -L1 < /proc/$$/environ        # 현재 셸의 환경 (NULL 구분)
xargs -0 -L1 < /proc/<other-PID>/environ   # 다른 프로세스 (보통 권한 필요)
```

## 흔한 함정

> [!WARNING]
> **cron 함정**: cron은 어떤 init 파일도 안 읽음. `PATH=/usr/bin:/bin` 정도만 set되어 `aws`·`psql`·`/usr/local/bin/*` 같은 명령이 "command not found". 해결: 스크립트에서 절대 경로 사용 또는 첫 줄에서 `PATH=...` 명시. SSH로 들어가서는 잘 동작하던 명령이 cron에서 안 되면 99% 이 함정이다.

셸 환경 모델은 미묘한 함정이 많다. 운영에서 실제로 부딪히는 종류만 정리한다.

가장 흔한 첫 함정은 SSH 진입 후 `.bashrc`가 안 읽혀서 alias가 안 먹는 현상이다. SSH는 login shell을 띄우므로 `.bash_profile`만 읽히는데, `.bash_profile`에서 `.bashrc`를 source하지 않으면 alias·함수가 다 누락된다. 새로 만든 사용자 계정에서 `.bash_profile`이 비어 있을 때 가장 자주 발생한다. 비슷한 맥락에서 non-interactive 셸에서는 alias가 기본적으로 안 먹는데, Bash가 non-interactive 셸에서 alias expansion을 끄기 때문이다 — 스크립트 안에서 alias를 정의해도 그 다음 줄에서 사용하려면 `shopt -s expand_aliases`를 먼저 set해야 한다.

운영 자동화의 가장 큰 함정은 cron의 PATH 부재다. cron은 `PATH=/usr/bin:/bin` 정도만 set하므로 `/usr/local/bin`이나 `~/.local/bin`에 설치된 도구는 모두 누락된다. `psql`이나 `aws` 같은 명령을 cron에서 호출하면 "command not found" 에러가 나는데, 같은 명령이 직접 셸에서는 잘 동작하니 디버깅이 어렵다. 해결은 (1) 스크립트에서 절대 경로 사용, (2) 스크립트 첫 줄에서 `PATH=...` 명시, (3) crontab 상단에 `PATH=...` 변수 정의 중 하나다. 비슷하게 `sudo`의 환경 reset도 자주 함정이 되는데, sudoers의 `env_reset` 기본 정책으로 거의 모든 환경 변수가 drop된다 — `--preserve-env`나 sudoers의 `env_keep` 명시가 필요하며, 보안상으로는 reset이 옳지만 "내 환경이 sudo에서 안 먹는다"는 혼란이 자주 생긴다.

시그니처가 미묘한 함정 중 하나가 `export VAR=$(cmd)` 패턴이다. `cmd`가 실패해도 export는 성공으로 처리되는데, `export` 자체의 exit code는 마지막 명령의 것이 아니라 자기 자신의 success(0)다. 그래서 `set -e`와 조합해도 명령 실패가 잡히지 않으며, 안전하게 쓰려면 `VAR=$(cmd) || exit; export VAR`처럼 분리해야 한다. 또 다른 운영 사고로 환경 변수 size 한계가 있는데, `MAX_ARG_STRLEN`(보통 128KB) 또는 `ARG_MAX`(보통 2MB) 초과 시 `execve`가 `E2BIG`로 실패한다. 환경 변수에 거대한 데이터(Base64 인코딩된 인증서 등)를 넣는 안티패턴이 만나는 한계다.

보안 측면에서는 PATH injection이 sudo로 실행되는 스크립트의 위협이다. 스크립트가 `cmd`처럼 상대 경로로 명령을 호출하면 공격자가 자기 디렉토리를 PATH 앞쪽에 두고 같은 이름의 트로이목마를 둘 수 있는데, sudoers의 `secure_path` 옵션으로 방어한다. 마지막으로 배포판별 default startup의 차이는 이식성 문제를 만드는데, Debian의 `/etc/skel/.bashrc`, RHEL의 `/etc/profile.d/`, macOS Bash 3.x의 다른 동작 — 모두 미묘한 차이가 있어 한 배포판에서 동작한 셸 설정이 다른 배포판에서 깨지는 일이 있다.

## B1-1 매핑

이번 과제는 환경 변수 설정의 정석을 직접 요구한다.

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

이렇게 두면 SSH 접속 시 모든 `AGENT_*` 변수가 자동으로 set된다. 명세에서 요구하는 "환경 변수로 실행 환경 고정"이 이 메커니즘의 직접 응용이다.

하지만 cron이 monitor.sh를 실행할 때는 이 환경을 받지 못한다는 점이 함정이다. cron은 non-interactive non-login으로 셸을 띄우므로 `.bash_profile`이 source되지 않는다. monitor.sh가 이 환경 변수들을 사용한다면 스크립트 안에서 직접 source해야 한다.

```bash
# monitor.sh 상단 권장 패턴
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
export LC_ALL=C
[ -f /home/agent-admin/.bash_profile ] && . /home/agent-admin/.bash_profile
```

`LC_ALL=C` 설정은 추가 안전장치다. cron 환경에는 `LANG`/`LC_ALL`이 없어서 `date`나 `ps` 같은 명령이 로케일에 따라 출력이 달라질 수 있다. `LC_ALL=C`로 명시하면 POSIX 표준 출력 형식을 강제할 수 있어, 스크립트의 파싱이 환경에 따라 깨지지 않는다.

## 인접 토픽

<details>
<summary><b>응용 토픽 — systemd Environment·/etc/environment·dotenv·direnv·secrets·shellcheck (펼치기)</b></summary>

셸 환경 모델의 응용은 환경 관리·보안·개발 편의의 세 축으로 정리해 볼 수 있다.

서비스 운영의 현대적 표준은 systemd unit의 `Environment=` / `EnvironmentFile=`이다. systemd로 관리되는 서비스는 셸 startup 파일에 의존하지 않고 unit 파일에서 환경을 명시하는데, 의존성·격리·관측이 모두 더 명시적이라 운영 가시성이 높아진다. 비슷한 맥락에서 PAM 기반의 `/etc/environment`는 모든 로그인에 set되는 환경 변수 파일로, 셸 종류와 무관하게 적용되며 KEY=VALUE 단순 형식만 지원한다.

개발자 편의 측면에서는 dotenv 패턴과 디렉토리별 환경 전환 도구가 자주 만난다. 프로젝트별 `.env` 파일에 환경 변수를 두고 코드가 로드하는 dotenv 패턴은 개발 편의는 좋지만 권한 관리·gitignore·secret 분리 등 보안 주의가 필수다. 한편 direnv·mise·asdf 같은 도구는 디렉토리별 자동 환경 전환을 제공하는데, 프로젝트 디렉토리에 들어가면 자동으로 그 환경(Node 버전, Python venv 등)이 set되고 나가면 복원된다. 멀티 프로젝트 작업 시 매우 유용하다.

보안 측면에서는 환경 변수의 secret 저장이 안티패턴인 이유를 짚어둘 가치가 있다. 환경 변수는 `/proc/PID/environ`으로 노출되고 child process에 자동 전파되므로 secret을 두기에 안전하지 않으며, HashiCorp Vault·AWS Secrets Manager·1Password CLI 같은 전용 도구를 사용하는 게 현대적 표준이다. 마지막으로 셸 스크립트를 다루는 모든 작업에 사실상 필수인 도구가 shellcheck로, 환경 변수 quote 누락·잘못된 비교·위험한 패턴 등을 자동 검출한다.

</details>

## 참고

- `man bash` — INVOCATION, FILES 섹션 (시작 동작의 정식 정의)
- `man 7 environ` — 환경 변수 모델
- `man 5 crontab` — cron 환경 모델
- `man 8 sudoers` — env_reset, env_keep 정책
- `man 5 systemd.exec` — systemd 환경 설정

---
출처: B1-1 (Layer 1.4) · 학습일: 2026-05-07
