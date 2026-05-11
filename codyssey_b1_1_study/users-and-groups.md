# 사용자·그룹·신원 (Identity)

> **TLDR** · B1-1 명세는 **3개 사용자(agent-admin/dev/test) + 2개 그룹(agent-common/core) + 특정 멤버십**을 요구. 핵심 사고 포인트는 `usermod -G` 함정 — 반드시 `-aG`(append) 명시. agent-test는 의도적으로 `agent-core`에서 빠져 production 자격 증명 접근 차단.

---

## 과제 요구사항

### 명세 명문 (B1-1 spec)

> **생성 계정**
> - `agent-admin` (운영/관리, cron 실행자)
> - `agent-dev` (개발/운영, monitor.sh 작성자)
> - `agent-test` (QA/테스트)
>
> **생성 그룹**
> - `agent-common`: admin, dev, test
> - `agent-core`: admin, dev

### 사용자 ↔ 그룹 관계도

```mermaid
graph LR
    A[agent-admin<br/>운영·cron 실행자]
    D[agent-dev<br/>개발자·monitor.sh 작성]
    T[agent-test<br/>QA]

    A -->|멤버| CM[agent-common<br/>공유 그룹 — upload_files RW]
    D -->|멤버| CM
    T -->|멤버| CM

    A -->|멤버| CO[agent-core<br/>핵심 그룹 — api_keys·log RW]
    D -->|멤버| CO

    style CM fill:#cce5ff
    style CO fill:#ffe6cc
    style T fill:#fff0f0
```

### 권한 의도 (누가 무엇에 접근?)

| 사용자 | upload_files (common 그룹) | api_keys (core 그룹) | /var/log/agent-app (core) |
|---|---|---|---|
| agent-admin | ✅ RW | ✅ RW | ✅ RW |
| agent-dev | ✅ RW | ✅ RW | ✅ RW |
| **agent-test** | ✅ RW | **❌ 차단** | **❌ 차단** |

→ **test는 production 자격 증명·로그 접근 차단** (보안 격리 의도)

### 검증 기준

```bash
id agent-admin                 # uid·gid·groups 모두 출력
id agent-dev
id agent-test
getent group agent-common      # 멤버: admin, dev, test
getent group agent-core        # 멤버: admin, dev
sudo -u agent-test ls /home/agent-admin/agent-app/api_keys
# → EACCES (Permission denied) 기대
```

---

## 구현 방법

### Step 1 — 그룹 생성 (멱등)

```bash
for group in agent-common agent-core; do
    if getent group "$group" >/dev/null 2>&1; then
        echo "[SKIP] group $group 이미 존재"
    else
        sudo groupadd "$group"
        echo "[OK] group $group 생성"
    fi
done
```

**기대 출력 (첫 실행)**:
```
[OK] group agent-common 생성
[OK] group agent-core 생성
```

### Step 2 — 사용자 생성

```bash
for user in agent-admin agent-dev agent-test; do
    if id "$user" >/dev/null 2>&1; then
        echo "[SKIP] user $user 이미 존재"
    else
        sudo useradd -m -s /bin/bash "$user"
        echo "[OK] user $user 생성"
    fi
done
```

옵션:
- `-m`: 홈 디렉토리(`/home/$user`) 자동 생성
- `-s /bin/bash`: 기본 셸 지정

### Step 3 — 그룹 멤버십 (★ `-aG` 필수)

```bash
sudo usermod -aG agent-common,agent-core agent-admin
sudo usermod -aG agent-common,agent-core agent-dev
sudo usermod -aG agent-common agent-test
```

> [!WARNING]
> `-a`(append) 빼면 기존 supplementary 그룹이 **모두 삭제**됨. 자동화에서 절대 사고. 항상 `-aG` 명시.

### Step 4 — 검증

```
$ id agent-admin
uid=1001(agent-admin) gid=1001(agent-admin) groups=1001(agent-admin),1002(agent-common),1003(agent-core)

$ id agent-test
uid=1003(agent-test) gid=1003(agent-test) groups=1003(agent-test),1002(agent-common)
                                                                                  ^^^
                                              ★ agent-core 없음 (의도된 격리)
```

### 멱등성 — 재실행 안전 흐름

```mermaid
flowchart TD
    A[setup/03 실행] --> B{그룹 존재?}
    B -->|예| C[SKIP]
    B -->|아니오| D[groupadd]
    C --> E{사용자 존재?}
    D --> E
    E -->|예| F[SKIP]
    E -->|아니오| G[useradd -m -s bash]
    F --> H[usermod -aG ...<br/>★ -aG라 항상 안전]
    G --> H
    H --> Z[완료]

    style C fill:#ffe6cc
    style F fill:#ffe6cc
    style D fill:#ccffcc
    style G fill:#ccffcc
    style H fill:#cce5ff
```

전체 스크립트: [setup/03-users-groups.sh](https://github.com/codewhite7777/codyssey_b1_1/blob/main/setup/03-users-groups.sh)

---

## 개념

### Linux 신원 모델 — `(uid, gid, supplementary GIDs)`

Linux의 모든 프로세스·파일은 신원을 가진다. 커널 내부에서는 다음 다층 구조.

```mermaid
graph TD
    P[프로세스 / 파일]
    P --> R["Real UID/GID<br/>로그인한 실제 사용자"]
    P --> E["Effective UID/GID<br/>★ 권한 검사에 사용<br/>setuid·sudo 시 변경"]
    P --> S["Saved UID/GID<br/>임시 drop/restore"]
    P --> SUP["Supplementary GIDs[]<br/>추가 그룹 (최대 65536개)"]

    style E fill:#ffe6cc
```

**왜 3종 UID 분리?** `passwd` 같은 setuid 프로그램에서 effective UID는 root(권한 행사), real UID는 호출자(감사 추적). 권한 위임의 다층 안전장치.

### Primary vs Supplementary Group

```
agent-admin
├── primary gid: agent-admin    ← 새 파일 만들면 이 그룹이 owner
└── supplementary: [agent-common, agent-core]  ← 기존 파일 접근 권한 검사
```

| | Primary | Supplementary |
|---|---|---|
| 개수 | 정확히 1개 | 0개 이상 |
| 저장 | `/etc/passwd` 4번째 필드 | `/etc/group` 멤버 목록 |
| 새 파일 소유 | ✅ 기본 group owner | ❌ 영향 X |
| 기존 파일 권한 | (owner 매칭 안 되면 검사) | ✅ 매칭되면 group 권한 |

★ 단, **setgid 디렉토리**(`/var/log/agent-app`)에서는 새 파일이 그 디렉토리의 그룹 상속 → primary 무시.

### `usermod -G` vs `-aG` 함정

```mermaid
sequenceDiagram
    participant U as agent-admin<br/>(현재 멤버: agent-common, agent-core)

    rect rgb(255, 220, 220)
    Note over U: ★ 위험: -G만 사용
    U->>U: usermod -G newgroup agent-admin
    Note over U: 결과: newgroup만<br/>(common, core 모두 사라짐!)
    end

    rect rgb(220, 255, 220)
    Note over U: ✓ 안전: -aG 사용
    U->>U: usermod -aG newgroup agent-admin
    Note over U: 결과: common, core, newgroup<br/>(기존 보존 + 추가)
    end
```

자동화 스크립트에서 가장 자주 만나는 사고. 항상 `-aG`.

### 그룹 변경의 비동기성

```mermaid
sequenceDiagram
    participant T1 as 터미널 1 (작업 중)
    participant K as 커널
    participant T2 as 터미널 2 (이미 열린 셸)

    T1->>K: usermod -aG agent-core agent-test
    K->>K: /etc/group 갱신 완료
    Note over T2: ★ 옛 GID set 유지<br/>(변경 미반영)

    Note over T1: "그룹 추가했으니 권한 됐겠지" 시도
    T1->>T2: agent-core 디렉토리 접근
    Note over T2: ❌ Permission denied<br/>(옛 GID로 검사)

    Note over T1,T2: 해결: 새 SSH 또는 newgrp agent-core로<br/>새 셸 띄우기
```

→ 그룹 추가 직후 같은 셸에서 즉시 권한 작업하면 실패. 새 세션 필요.

---

## 인접 토픽 (선택)

<details>
<summary><b>고급 — capabilities · user namespace · PAM · NSS · systemd-homed (펼치기)</b></summary>

**POSIX capabilities** — root 권한을 39+개 단위로 쪼갠 메커니즘. setuid의 안전한 대안 (`cap_net_bind_service`로 1024 미만 포트 바인딩만 허용 등).

**User namespace** — 컨테이너 안 root가 호스트의 일반 사용자로 매핑되는 격리. `/proc/PID/uid_map`. rootless container의 핵심.

**PAM (Pluggable Authentication Modules)** — 인증 단계의 모듈식 구성. SSH·sudo·screen unlock이 모두 PAM 경유. 2FA·패스워드 정책이 PAM 모듈로 구현.

**NSS (Name Service Switch)** — 사용자·그룹 lookup의 추상화. `/etc/nsswitch.conf`로 파일·LDAP·SSSD 등 백엔드 순서 정의. 회사 환경의 LDAP 사용자가 `getent`로 보이는 이유.

**systemd-homed** — LUKS 암호화된 portable 홈으로 사용자 관리. `/etc/passwd` 의존도 ↓, 노트북 도난 시 데이터 보호.

</details>

---

## 참고

- `man 5 passwd`, `man 5 group`, `man 5 shadow`
- `man useradd`, `man usermod`, `man getent`
- `man 7 credentials` — UID/GID 모델 전반
- 관련 노트: [file-permissions.md](./file-permissions.md) (권한 검사 알고리즘)
- 관련 노트: [shell-environment.md](./shell-environment.md) (그룹 변경의 셸 환경 영향)

---
출처: B1-1 (Layer 1.2) · 학습일: 2026-05-12
