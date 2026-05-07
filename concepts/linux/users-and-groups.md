# 사용자·그룹·신원 (Identity)

## 개요
Linux의 멀티유저 권한 모델은 모든 프로세스·파일에 부여되는 `(uid, gid)` 쌍과, 사용자가 속할 수 있는 supplementary group 집합으로 표현된다. 권한 검사는 syscall 진입 시점에 커널이 호출자의 신원과 대상 객체의 권한 메타데이터를 비교하여 수행된다.

## 왜 알아야 하나
- 다중 사용자 격리·권한 제어의 기반 (POSIX 권한, ACL, capabilities, namespace 모두 이 위에 쌓임)
- 보안 사고의 다수가 잘못된 신원 운영 (root 남용, 공유 계정, privilege escalation)
- 컨테이너의 user namespace, K8s의 ServiceAccount, sudo 정책 모두 이 모델의 확장

## 신원의 구성 요소

```
프로세스 신원 (커널의 cred struct)
   ├─ Real UID/GID         로그인 사용자
   ├─ Effective UID/GID    실제 권한 검사에 사용 (setuid 시 변경)
   ├─ Saved UID/GID        권한 임시 drop/restore 위해 보존
   ├─ Filesystem UID/GID   파일 접근 시 — 보통 effective와 동일 (NFS legacy)
   └─ Supplementary GIDs[] 추가 그룹 집합 (권한 검사 시 모두 매칭 시도)
```

**setuid 비트** 또는 `seteuid()` 시스템 콜로 effective UID가 바뀌는 게 sudo·passwd 같은 특권 동작의 메커니즘.

## Primary vs Supplementary Group

| 구분 | 의미 | 동작 |
|---|---|---|
| **Primary GID** | `/etc/passwd` 4번째 필드 | 새 파일 생성 시 기본 group owner (단, setgid 디렉토리 예외) |
| **Supplementary GIDs** | `/etc/group`에서 멤버 등록 | 권한 검사 시 group 비트와 매칭. NGROUPS_MAX(보통 65536) 제한 |

```
사용자 agent-admin
  ├─ uid: 1001
  ├─ primary gid: 1001 (agent-admin)
  └─ supplementary: [1002 (agent-common), 1003 (agent-core)]
```

→ `agent-admin`이 만든 새 파일의 group owner는 기본 `agent-admin`(primary). 단 setgid 디렉토리에서는 그 디렉토리의 group을 상속.

## 정보 저장 위치

```
/etc/passwd     사용자 (이름, uid, primary gid, GECOS, home, shell)
/etc/shadow     비밀번호 해시 + 만료 정책 (root만 read, 0640 root:shadow)
/etc/group      그룹 (이름, gid, 멤버 목록)
/etc/gshadow    그룹 패스워드 (거의 안 씀)
```

`/etc/passwd` 한 줄:
```
agent-admin:x:1001:1001:Agent Admin:/home/agent-admin:/bin/bash
└──┬──────┘ │ └┬─┘ └┬─┘ └────┬───┘ └─────┬───────┘ └────┬───┘
   name     │ uid  pgid    GECOS         home        login shell
            password placeholder ('x' = use shadow)
```

**NSS (Name Service Switch)**: 이 파일 외에 LDAP·SSSD·systemd-homed 같은 백엔드도 조회 가능 (`/etc/nsswitch.conf`로 우선순위 정의).

## 한 번 보자

```bash
id                                  # 현재 신원 (uid/gid/groups)
id -G                               # supplementary GID만 (수치)
id -Gn                              # supplementary 이름

getent passwd agent-admin           # NSS 통해 조회 (LDAP 등 포함)
getent group agent-core

# 그룹 멤버십 변경 (즉시 반영 X — 새 세션에서 적용)
sudo usermod -aG agent-core agent-dev    # ★ -a 누락 시 기존 supplementary 다 날림
sudo gpasswd -a agent-dev agent-core     # 동일 동작 (gpasswd 사용)

newgrp agent-core                   # 새 셸에서 primary group 일시 변경

# setuid/setgid 확인
ls -l /usr/bin/passwd               # -rwsr-xr-x ← s가 setuid (root 권한으로 실행)
ls -l /usr/bin/wall                 # setgid tty 예시

# 임의 프로세스 신원 보기
cat /proc/$$/status | grep -E 'Uid|Gid|Groups'
```

## 흔한 함정

- **`usermod -G` 함정** — `-a` 없이 사용 시 기존 supplementary 다 삭제. 자동화 스크립트에서 치명적.
- **그룹 변경 즉시 반영 X** — 커널이 프로세스의 GID set을 `setgroups()` 시점에만 업데이트. 기존 셸은 옛 GID로 실행 중. `newgrp`나 재로그인 필요.
- **NGROUPS_MAX 한계** — 사용자가 너무 많은 그룹에 속하면 NFS·일부 도구가 깨짐 (전통적 16개 제한 잔재).
- **setuid + 셸 스크립트** — 대부분 OS가 무시 (race condition 보안 이슈). 컴파일된 바이너리만 setuid 작동.
- **`sudo` ≠ root** — sudoers 정책 기반의 effective UID 변경. 감사 로그(`/var/log/auth.log`)에 기록. 권한 분리 + 추적성.
- **컨테이너의 root** — user namespace 매핑으로 "컨테이너 내 root = 호스트 일반 사용자"일 수 있음. 보안 가정이 깨질 수 있음.
- **GECOS 필드의 콤마** — `,`로 구분된 sub-field (full name, room, work phone, home phone). 잘못 쓰면 chfn 깨짐.

## B1-1 매핑

```
주민          primary gid     supplementary
agent-admin   agent-admin     agent-common, agent-core
agent-dev     agent-dev       agent-common, agent-core
agent-test    agent-test      agent-common
```

핵심 의도:
- **agent-test가 agent-core에서 빠짐** → `api_keys/`, `/var/log/agent-app/` 접근 차단 (group 권한이 ONLY core)
- monitor.sh를 **agent-dev 소유, agent-core 그룹**으로 두면 → admin이 cron으로 실행 가능 (admin도 core 멤버)

## 인접 토픽

- **POSIX capabilities** — root 권한을 39+개 단위로 쪼갠 것. setuid의 대안 (`cap_net_bind_service`, `cap_sys_admin` 등)
- **PAM (Pluggable Auth Modules)** — 인증 단계의 모듈식 구성 (`/etc/pam.d/`)
- **NSS** — 사용자·그룹 lookup의 추상화 layer
- **user namespace** — 컨테이너가 자체 uid 매핑을 갖는 메커니즘 (`/proc/PID/uid_map`)
- **systemd-homed** — 사용자를 portable home (LUKS encrypted) 형태로 관리하는 새 모델
- **ACL (POSIX 1003.1e)** — owner/group/other로 부족할 때 (다음 학습)

## 참고
- `man 5 passwd`, `man 5 shadow`, `man 5 group`
- `man useradd`, `man usermod`, `man getent`
- `man 7 credentials` — UID/GID 모델 전반
- `man 7 capabilities` — 인접

---
출처: B1-1 (Layer 1.2) · 학습일: 2026-05-07
