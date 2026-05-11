# 안전한 Bash 스크립트: set -euo pipefail

> **TLDR** · `set -e`(에러 시 즉시 종료), `set -u`(unset 변수 사용 시 에러), `set -o pipefail`(파이프 중간 실패 감지)의 3종 세트가 운영 스크립트의 안전 표준. `IFS=$'\n\t'`도 함께 두면 공백·glob 함정 추가 방어. 미세한 함정도 있지만 대부분 스크립트가 이 패턴으로 시작.

## 개요

기본 Bash는 매우 관대해서 에러가 발생해도 다음 줄로 그냥 넘어간다. 한 줄이 실패해도 전체 스크립트는 success(0)으로 끝날 수 있다. 운영 자동화에서는 치명적인 행동이다 — setup 스크립트가 중간에 실패했는데 "성공"으로 끝나면 production이 깨진 상태로 남는다.

`set -euo pipefail`은 이 관대함을 엄격하게 만드는 한 줄이다. 1990년대부터 운영 자동화 커뮤니티가 표준으로 정착시킨 패턴이며, 현대 CI/CD·설치 스크립트의 거의 모든 첫머리에 있다.

## 왜 알아야 하나

운영 스크립트가 silent failure로 production을 망가뜨리는 사고가 흔하다. 디스크 백업 스크립트가 중간 명령 실패를 무시하고 "백업 완료"를 보고하거나, 배포 스크립트가 build 실패해도 deploy를 진행하거나, setup 스크립트가 일부만 적용된 채로 끝나는 경우 등이다.

`set -euo pipefail`은 이런 사고를 방지하는 가장 단순하고 효과적인 도구다. 단 미세한 함정이 있어 모든 케이스에 만능은 아니다 — 의도된 실패(예: `grep -q`가 못 찾았을 때)를 처리하는 패턴도 함께 알아야 한다.

## set -e (errexit)

명령이 non-zero exit code를 반환하면 즉시 스크립트 종료.

```bash
#!/bin/bash
set -e
echo "step 1"
false             # exit code 1 → 여기서 스크립트 종료
echo "step 2"     # 실행 안 됨
```

이 한 줄이 silent failure를 거의 막는다. 하지만 의도된 실패(grep, 조건 검사 등)도 종료시키므로 우회 패턴이 필요하다.

```bash
# 의도된 실패 우회 패턴
grep "pattern" file || true       # || true로 항상 success로
result=$(grep "pattern" file) || result=""   # 또는 default 값
if grep -q "pattern" file; then   # if 안에서는 set -e 비활성
    ...
fi
```

`set -e`의 미세한 함정: 명령이 `if`·`while`·`&&`·`||`의 일부로 실행될 때는 비활성화된다. 함수 안에서도 일부 케이스 (legacy bash 동작).

## set -u (nounset)

unset 변수를 사용하면 즉시 에러.

```bash
#!/bin/bash
set -u
echo "$UNDEFINED_VAR"     # bash: UNDEFINED_VAR: unbound variable → 종료
```

타이포나 변수명 오류를 잡는 데 매우 유용하다. 단 의도적으로 unset일 수 있는 변수는 default 값 처리.

```bash
echo "${OPTIONAL_VAR:-default}"   # unset이면 default
echo "${OPTIONAL_VAR:-}"          # unset이면 빈 문자열
```

`$@`·`$*`는 인자가 없으면 unset이라 `set -u`에 걸린다. `"${@:-}"` 패턴으로 우회.

## set -o pipefail

파이프라인에서 중간 명령의 실패를 감지.

```bash
#!/bin/bash
set -e
false | true              # 파이프 전체 exit code는 마지막 명령(true)의 0
echo "여기 도달함"        # set -e가 못 잡음!

set -o pipefail
false | true              # 이제 pipefail이 1 반환 → set -e 발동
echo "여기 도달 안 함"
```

운영에서 매우 중요한 옵션이다. `cmd | grep | sort` 같은 파이프에서 첫 명령이 실패해도 grep이 빈 입력으로 0 반환, sort도 0 반환 → 전체 success로 판단되어 silent failure 발생.

## 통합 패턴: set -euo pipefail

세 옵션을 한 줄로:

```bash
#!/usr/bin/env bash
set -euo pipefail
```

순서·표기는 자유 — `set -e -u -o pipefail`도 동일.

추가로 IFS를 안전한 값으로 설정하는 패턴도 흔히 쓴다.

```bash
#!/usr/bin/env bash
set -euo pipefail
IFS=$'\n\t'                # 공백을 separator에서 제외 → glob 함정 ↓
```

이 4줄이 운영 스크립트의 표준 헤더다.

## 동작 흐름

```mermaid
flowchart TD
    A[명령 실행] --> B{exit code == 0?}
    B -->|Yes| C[다음 명령 진행]
    B -->|No| D{if/while/&&/||의<br/>일부인가?}
    D -->|Yes| C
    D -->|No| E{set -e 활성?}
    E -->|Yes| F["★ 스크립트 즉시 종료<br/>(non-zero exit)"]
    E -->|No| C

    G[변수 사용] --> H{변수 unset?}
    H -->|No| C
    H -->|Yes| I{set -u 활성?}
    I -->|Yes| F
    I -->|No| J["빈 문자열로 평가<br/>(silent)"]

    K[파이프 실행] --> L{중간 명령 실패?}
    L -->|No| C
    L -->|Yes| M{pipefail 활성?}
    M -->|Yes| F
    M -->|No| C

    style F fill:#ffcccc
    style C fill:#ccffcc
```

## 한 번 보자

```bash
#!/usr/bin/env bash
set -euo pipefail

# 의도된 실패 처리
if grep -q "pattern" /etc/hosts; then
    echo "found"
fi

# || true로 의도된 실패 허용
result=$(grep "maybe_missing" file || true)

# default 값으로 unset 우회
NAME="${USER_NAME:-anonymous}"
echo "Hello, $NAME"

# 파이프 안전성 검증
if false | grep "foo" || true; then
    echo "이건 도달"
fi
```

`set -x`로 trace 모드를 추가하면 각 명령 실행을 stderr에 출력하므로 디버깅에 유용:

```bash
set -euxo pipefail        # x 추가 → trace
```

또는 일부 구간만:

```bash
set -x
risky_command
set +x                    # trace 비활성
```

## 흔한 함정

> [!WARNING]
> **`set -e`의 미세한 함정**: `if`·`&&`·`||`·`while`·함수의 일부로 실행되는 명령에는 적용 안 됨. 함수 안에서 `set -e`가 항상 작동한다고 가정하면 함정. 명시적으로 exit code 체크하는 게 안전.

`set -e`의 가장 미묘한 함정은 컨텍스트별 동작 차이다. 함수 안에서 명령이 실패해도 함수 자체는 마지막 명령의 exit code를 반환하므로, 호출자가 그 exit code를 체크하지 않으면 사일런트 실패가 가능하다. Bash 4.4+에서 `shopt -s inherit_errexit`로 일부 개선되었지만 완벽하지 않다.

`pipefail`도 모든 케이스에 만능은 아니다. 예를 들어 `head -1`이 입력을 1줄만 읽고 종료하면, 그 앞의 generator가 SIGPIPE로 종료되며 pipefail이 잘못 트리거된다. `set +o pipefail`로 일시적으로 끄거나 `(cmd | head -1) || true` 패턴을 사용.

`set -u`는 array 동작에서 함정이 있다. `${array[@]}`가 빈 array일 때 unbound로 잡히는 경우가 있어 `"${array[@]+"${array[@]}"}"` 같은 복잡한 패턴이 필요할 수 있다. 대부분의 경우 default 값 처리(`${var:-}`)로 우회.

대화형 셸에서는 절대 `set -e`를 켜지 말 것. 명령 실패 시 셸이 종료되어 세션이 끊긴다 (`.bashrc`에 `set -e` 절대 금지).

## B1-1 매핑

monitor.sh의 안전 헤더:

```bash
#!/usr/bin/env bash
# monitor.sh — 시스템 관제 자동화

set -euo pipefail
IFS=$'\n\t'

# 의도된 실패 처리 (pgrep이 못 찾을 수 있음)
PID=$(pgrep -f 'agent_app\.py' | head -1 || true)
if [ -z "$PID" ]; then
    echo "[ERROR] agent_app.py not running"
    exit 1
fi

# 정상 흐름
echo "[OK] PID=$PID"
```

setup 스크립트의 안전 헤더:

```bash
#!/usr/bin/env bash
# setup/02-firewall.sh

set -euo pipefail

# 멱등성 — 이미 활성화되어 있어도 안전
sudo ufw --force reset           # --force로 confirm 우회
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 20022/tcp
sudo ufw allow 15034/tcp
sudo ufw --force enable

echo "[OK] firewall configured"
```

`set -e`가 활성화되어 있으므로 어느 명령이 실패해도 즉시 종료된다. 부분적으로 설정되는 상태를 방지.

verify.sh에서는 의도적으로 일부 실패를 허용:

```bash
#!/usr/bin/env bash
set -uo pipefail              # ★ -e는 빼기 (실패한 검증도 끝까지 진행)

failures=0

check() {
    if ! eval "$1"; then
        echo "[FAIL] $2"
        ((failures++)) || true
    else
        echo "[OK] $2"
    fi
}

check 'systemctl is-active sshd' 'SSH 데몬 실행'
check 'ss -tulnp | grep -q :20022' 'SSH 포트 20022 LISTEN'
check 'ss -tulnp | grep -q :15034' '앱 포트 15034 LISTEN'

[ $failures -eq 0 ]            # 마지막에 전체 결과로 종료
```

## 인접 토픽

<details>
<summary><b>응용 토픽 — ERR trap·shopt·inherit_errexit·strict mode (펼치기)</b></summary>

`trap 'echo "error at line $LINENO"' ERR`로 에러 발생 시 핸들러를 실행할 수 있다. `set -e`와 결합해 어느 줄에서 실패했는지 로깅. 디버깅·alerting에 유용.

`shopt -s inherit_errexit`는 Bash 4.4+ 옵션으로, 서브셸에서도 `set -e`가 상속되도록 한다. 기본은 inherit 안 되어 `result=$(failing_cmd)`가 set -e에 안 걸리는 함정 해결.

unofficial "Unofficial Bash Strict Mode"는 Aaron Maxwell이 제안한 4줄 헤더 — `set -euo pipefail`, `IFS=$'\n\t'`, `shopt -s inherit_errexit`. 운영 표준으로 자리잡았다.

`set -E`(errtrace)는 `ERR` trap이 함수·서브셸·command substitution까지 상속되게 한다. trap 활용을 깊이 할 때 필요.

bash 외 셸에서의 차이도 알아둘 가치가 있다. zsh의 `EMULATE_BASH` 모드는 거의 호환이지만 set -e 동작이 약간 다르다. fish는 set -e 대신 다른 모델 (status 명시 체크).

</details>

## 참고

- `man bash` — SHELL BUILTIN COMMANDS의 `set`, `shopt`
- [BashFAQ/105: set -e](https://mywiki.wooledge.org/BashFAQ/105) — set -e의 모든 함정
- [Aaron Maxwell — Bash Strict Mode](http://redsymbol.net/articles/unofficial-bash-strict-mode/)

---
출처: B1-1 (Layer 4.2) · 학습일: 2026-05-11
