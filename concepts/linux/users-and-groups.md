# 사용자와 그룹

## 한 줄 정의
리눅스의 다중 사용자 보안 모델 — 모든 파일·프로세스에 **소유자(user)**와 **그룹(group)**이 부여되어 누가 무엇을 할 수 있는지 통제하는 기반.

## 왜 필요한가

같은 머신을 여러 사람·서비스가 쓸 때:
- A의 파일을 B가 함부로 못 읽게
- 운영팀(admin)과 개발팀(dev)에게 다른 권한
- 앱이 자기 데이터만 건드리고 다른 건 못 건드리게 (격리)

이 과제(B1-1)에서:
- `agent-admin`(운영) / `agent-dev`(개발) / `agent-test`(QA) → 역할 분리
- `agent-common`(전원) / `agent-core`(admin+dev) → 데이터 접근 분리

## 핵심 원리

### 사용자(User)
- **UID**(User ID, 정수)로 식별 — 이름은 사람용, OS는 UID로 처리
- root는 UID=0 (모든 권한)
- 일반 사용자는 보통 UID >= 1000

### 그룹(Group)
- **GID**(Group ID)로 식별
- 한 사용자는 **여러 그룹**에 속할 수 있음

### Primary vs Supplementary Group

```
사용자 agent-admin
  ├─ Primary Group: agent-admin (보통 사용자명과 같은 그룹이 자동 생성됨)
  └─ Supplementary Groups: agent-common, agent-core
```

| 구분 | Primary | Supplementary |
|---|---|---|
| 개수 | **항상 정확히 1개** | 0개 이상 |
| 새 파일 만들 때 | 이 그룹이 기본 소유 그룹 | 영향 없음 (단 ACL/setgid 제외) |
| 저장 위치 | `/etc/passwd` (각 사용자 행에 GID로) | `/etc/group` (그룹별 멤버 목록) |
| 변경 명령 | `usermod -g <group>` | `usermod -aG <group>` |

### 정보가 저장되는 파일
```
/etc/passwd   사용자 목록 (이름, UID, primary GID, 홈디렉토리, 셸)
/etc/group    그룹 목록 (이름, GID, 멤버 사용자들)
/etc/shadow   암호 해시 (root만 읽기 가능)
```

`/etc/passwd` 한 줄 예시:
```
agent-admin:x:1001:1001:Agent Admin:/home/agent-admin:/bin/bash
└─────┬────┘ │ │    │   └────┬────┘ └──────┬──────┘ └────┬────┘
   이름      │ UID  │      설명           홈              로그인 셸
             └─암호 └─primary GID
             ('x' = 실제 해시는 /etc/shadow에)
```

## 자주 쓰는 명령

```bash
# 현재 사용자 + 그룹 확인
id
# 출력: uid=1001(agent-admin) gid=1001(agent-admin) groups=1001(agent-admin),1002(agent-common),1003(agent-core)

# 특정 사용자 정보
id agent-admin

# 사용자 생성 (홈 디렉토리 + 기본 셸 지정)
sudo useradd -m -s /bin/bash agent-admin
# -m: 홈 만들기 (/home/agent-admin/)
# -s: 로그인 셸 지정

# 비밀번호 설정
sudo passwd agent-admin

# 그룹 생성
sudo groupadd agent-common
sudo groupadd agent-core

# 사용자를 그룹에 추가 (supplementary)
sudo usermod -aG agent-common agent-admin
# -a (append): 기존 그룹 유지하고 추가
# -G (groups): supplementary group 지정
# ★ -a 빼먹으면 기존 supplementary 다 날리고 새것만 됨 (자주 하는 실수!)

# 사용자가 속한 그룹 확인
groups agent-admin

# 그룹의 멤버 확인
getent group agent-core
```

## 흔한 오해

- ❌ `usermod -G newgroup user` → 이러면 user의 기존 supplementary group **다 사라짐**. **`-aG` 써야 추가됨**.
- ❌ "primary group과 사용자명은 항상 같다" → 보통 자동으로 그렇게 만들어지지만 아닐 수도 있음 (`useradd -g existing_group`).
- ❌ "그룹 변경하면 즉시 반영" → ✅ 새 셸/SSH 접속해야 적용. 기존 셸은 옛 그룹 보유. (`newgrp <group>`로 새 셸 띄우면 즉시 적용)
- ❌ "root와 sudo는 같다" → 비슷하지만 다름. sudo는 "root처럼 행동할 수 있는 권한 위임" + 감사로그 남음.
- ❌ "agent-test 사용자는 agent-core가 아니니까 코어 데이터 못 봄" → ✅ 맞지만 단, **공유 디렉토리에 그 파일이 있으면** 디렉토리 권한에 따라 우회될 수도 있음 → 다음 노트 [file-permissions.md](./file-permissions.md) 참조

## 이번 과제 매핑

```
사용자          Primary Group     Supplementary Groups
─────────────  ────────────────  ─────────────────────────────
agent-admin    agent-admin       agent-common, agent-core
agent-dev      agent-dev         agent-common, agent-core
agent-test     agent-test        agent-common
```

→ `agent-test`는 `agent-core`에 없음 → `api_keys/`, `/var/log/agent-app/` 접근 불가 (이게 명세의 의도)

## 관련 개념
- [file-permissions.md](./file-permissions.md) — group 권한이 실제로 어떻게 작용하는지
- (예정) acl.md — owner/group/other로 부족할 때 (B1-1 핵심)

## 출처
- B1-1 (Layer 1.2)
- 학습일: 2026-05-07
- 참고: `man useradd`, `man usermod`, `man groups`, `man 5 passwd`
