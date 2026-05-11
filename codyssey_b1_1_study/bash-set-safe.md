# 안전한 Bash — set -euo pipefail

> **한 줄로** · 기본 Bash는 명령이 실패해도 다음 줄로 그냥 넘어가요. `set -euo pipefail` 한 줄로 **"에러 나면 즉시 멈춤"**, **"빈 변수 쓰면 에러"**, **"파이프 중간 실패도 감지"** 3가지를 모두 켤 수 있어요. B1-1 monitor.sh·setup 스크립트의 표준 헤더.

---

## 과제 요구사항

### 이게 무슨 작업?

회사 비유:
- 일반 Bash = **"문제 생겨도 계속 진행하는 직원"** → 사일런트 실패
- `set -euo pipefail` = **"문제 생기면 즉시 보고하고 중단하는 직원"** → 안전

setup 스크립트가 중간에 실패했는데 그대로 진행하면 **production이 깨진 상태로 남음**. 안전 모드는 이를 막아요.

### 명세 원문 (원본 그대로)

> **권장 헤더**
> ```bash
> #!/usr/bin/env bash
> set -euo pipefail
> ```
>
> **자기평가**
> - `set -euo pipefail`을 사용한 이유를 설명할 수 있다

### 무엇을 켜나

| 옵션 | 의미 |
|---|---|
| `set -e` | **에러 발생 시 즉시 종료** (silent failure 방지) |
| `set -u` | **unset 변수 사용 시 에러** (타이포 방지) |
| `set -o pipefail` | **파이프 중간 명령 실패 감지** |

### 잘 됐는지 확인하기

```bash
# 안전 모드 검증
cat <<'EOF' > /tmp/test.sh
#!/usr/bin/env bash
set -euo pipefail
echo "step 1"
false           # 실패 → 종료
echo "step 2"   # 도달 안 함
EOF
chmod +x /tmp/test.sh
/tmp/test.sh
echo $?         # 1 (안전 모드 동작)
```

`step 2`가 출력 안 되면 정상.

---

## 구현 방법

### Step 1 — 모든 스크립트의 첫 줄

```bash
#!/usr/bin/env bash
set -euo pipefail
```

`set -e -u -o pipefail`로 풀어 써도 동일.

### Step 2 — 의도된 실패 처리 (★ 필수)

`set -e`는 모든 실패에 반응. 의도적으로 실패할 수 있는 명령은 우회 패턴 필요.

```bash
# 1. || true로 실패 허용
grep "pattern" file || true

# 2. 결과를 변수에 + default
result=$(grep "pattern" file || echo "")

# 3. if 안에서는 set -e 자동 비활성
if grep -q "pattern" file; then
    echo "found"
fi
```

### Step 3 — unset 변수에 default 값

`set -u`는 빈 변수도 잡음. 의도적으로 비어있을 수 있는 변수는 default 처리.

```bash
# unset이면 default 사용
echo "${OPTIONAL_VAR:-default}"

# unset이면 빈 문자열
echo "${OPTIONAL_VAR:-}"

# 환경 변수 default
AGENT_HOME="${AGENT_HOME:-/home/agent-admin/agent-app}"
```

### Step 4 — IFS도 안전하게 (보너스)

```bash
#!/usr/bin/env bash
set -euo pipefail
IFS=$'\n\t'
```

`IFS`(Internal Field Separator)는 word-split할 때 구분자. 기본은 공백·탭·개행. `\n\t`만 두면 공백 함정이 더 줄어들어요.

### Step 5 — verify.sh의 예외 — `-e` 빼기

검증 스크립트는 **하나가 실패해도 계속 진행**해야 종합 결과를 볼 수 있어요. 그래서 `-e`만 빼는 게 정석:

```bash
#!/usr/bin/env bash
set -uo pipefail          # -e 없음 (의도)

failures=0

check() {
    if eval "$1"; then
        echo "[OK] $2"
    else
        echo "[FAIL] $2"
        failures=$((failures + 1))
    fi
}

check 'systemctl is-active ssh' 'sshd 동작'
check 'ss -ltn | grep -q :20022' 'SSH 포트 20022'
check 'ss -ltn | grep -q :15034' 'agent 포트 15034'

# 마지막에 종합 결과로 종료
[ $failures -eq 0 ]
```

전체 구현: [setup/verify.sh](https://github.com/codewhite7777/codyssey_b1_1/blob/main/setup/verify.sh)

---

## 개념

### `set -e` 흐름

```mermaid
flowchart LR
    A[명령 실행] --> B{exit code 0?}
    B -->|예| C[다음 명령]
    B -->|아니오| D{if/while/&&/||?}
    D -->|예| C
    D -->|아니오| E["★ 즉시 스크립트 종료"]

    style E fill:#ffcccc
    style C fill:#ccffcc
```

`if`·`while`·`&&`·`||`의 일부로 실행되는 명령은 실패해도 `set -e`가 무시. 의도된 실패 처리 패턴.

### `set -u` — unset 변수 잡기

```bash
set -u

# ❌ 타이포 - 잡힘
echo "$AGNET_HOME"      # bash: AGNET_HOME: unbound variable → 종료

# ✅ default
echo "${AGNET_HOME:-/default}"
```

타이포·미정의 변수를 즉시 탐지. 운영에서 매우 유용.

### `set -o pipefail` — 파이프 중간 실패 잡기

```bash
# pipefail 없이
false | true
echo $?         # 0 (마지막 명령 true의 exit code)

# pipefail 있으면
set -o pipefail
false | true
echo $?         # 1 (★ 중간 실패 잡힘)
```

`cmd | grep | sort` 같은 파이프에서 첫 명령이 실패해도 grep·sort가 0 반환하면 전체 success로 판단 → 사일런트 실패. pipefail이 이를 막아요.

### 안전 모드의 효과 (정리)

| 옵션 | 잡는 함정 |
|---|---|
| `set -e` | 중간 명령 실패 무시 |
| `set -u` | 타이포·미정의 변수 |
| `set -o pipefail` | 파이프 중간 실패 |
| `IFS=$'\n\t'` | 공백·glob 함정 |

### 의도된 실패 패턴 (★ 가장 중요)

```bash
set -e

# ❌ set -e가 잡음
pgrep agent-app                    # 없으면 exit 1 → 스크립트 종료

# ✅ || true로 의도된 실패 허용
PID=$(pgrep agent-app || true)
if [ -z "$PID" ]; then
    echo "[ERROR] agent-app 없음"
    exit 1
fi

# ✅ if 안에서는 자동 비활성
if pgrep -q agent-app; then
    echo "[OK] agent-app 동작"
fi
```

### `set -x` — 트레이스 모드 (디버깅)

```bash
set -x          # 실행되는 모든 명령을 stderr에 출력
risky_command
set +x          # trace 끔
```

또는 처음부터:
```bash
set -euxo pipefail        # x 추가 → trace 모드
```

디버깅에 매우 유용. production에서는 끔.

### `set -e`의 함정들

```mermaid
flowchart LR
    A[명령 실패] --> B{어느 컨텍스트?}
    B -->|단독 실행| C["✅ set -e 잡음"]
    B -->|if/while 안| D["★ 비활성 (의도)"]
    B -->|&&/||의 일부| D
    B -->|함수 안| E["⚠ 미묘 (Bash 4.4+ 일부 개선)"]
    B -->|서브셸 안| F["⚠ inherit_errexit 옵션 필요"]

    style C fill:#ccffcc
    style D fill:#cce5ff
    style E fill:#ffe6cc
    style F fill:#ffe6cc
```

대부분의 경우 잘 동작하지만, **함수**·**서브셸**에서는 미묘한 함정 있음. 안전을 위해:

```bash
shopt -s inherit_errexit    # Bash 4.4+ — 서브셸에도 set -e 상속
```

### 대화형 셸에서는 절대 set -e 금지

`.bashrc`에 `set -e` 두면 명령 하나 실패해도 셸이 종료됨 (세션 끊김). 스크립트 안에서만 사용.

### B1-1 표준 헤더 정리

monitor.sh와 모든 setup 스크립트는 이 헤더로 시작:

```bash
#!/usr/bin/env bash
set -euo pipefail

# 의도된 실패 처리
PID=$(pgrep -f "agent-app" 2>/dev/null || true)
if [ -z "$PID" ]; then
    echo "[ALERT] agent-app 미실행"
    exit 1
fi

# 환경 변수 default
LOG_FILE="${AGENT_LOG_DIR:-/var/log/agent-app}/monitor.log"
```

verify.sh만 예외 — `-e`만 빼서 모든 검증을 끝까지 진행.

---

## 참고

- `man bash` — SHELL BUILTIN COMMANDS의 `set`
- [Bash Strict Mode](http://redsymbol.net/articles/unofficial-bash-strict-mode/) — Aaron Maxwell의 표준
- 관련 노트: [bash-fundamentals.md](./bash-fundamentals.md) — Bash 기본
- 관련 노트: [bash-trap.md](./bash-trap.md) — 에러 처리 핸들러

---
출처: B1-1 (Layer 4.2) · 학습일: 2026-05-12
