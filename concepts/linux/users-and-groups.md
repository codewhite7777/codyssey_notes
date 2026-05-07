# 사용자와 그룹 = 단지 주민과 동호회

## 한 마디로
컴퓨터를 같이 쓰는 사람들을 **주민(사용자)**과 **모임(그룹)**으로 묶어서 누가 어디 들어가고 뭘 할 수 있는지 관리.

## 왜 알아야 해?

같은 컴퓨터를 여러 사람이 쓸 때:
- 내 폴더를 남이 함부로 못 보게
- 운영팀과 개발팀에게 다른 권한
- 앱이 자기 데이터만 만지고 다른 건 못 만지게

이 과제에서:
- 운영(`agent-admin`), 개발(`agent-dev`), 테스트(`agent-test`) — 역할마다 다른 주민
- 둘 이상이 함께 쓸 폴더는 모임(그룹)으로 묶음

## 그림으로 보기

```
              단지 주민 명단
─────────────────────────────────────────────
주민          본인 호수       가입한 동호회
─────         ─────────       ─────────────────────
agent-admin   agent-admin     [헬스, 운영팀]
agent-dev     agent-dev       [헬스, 운영팀]
agent-test    agent-test      [헬스]

[헬스]   동호회 = agent-common  (모두 함께 쓰는 공유 공간)
[운영팀] 동호회 = agent-core    (중요한 일만 하는 사람)
```

→ `agent-test`는 [운영팀] 동호회 아님 → 운영팀 전용 자료실 못 들어감.

## 핵심 개념 단 두 가지

### 1. 한 사람은 모임 여러 개에 들 수 있음

```
  agent-admin
       ├─ 본인 호수 (primary group)        ← 항상 1개
       └─ 추가 가입 모임 (supplementary)    ← 0개 이상
              ├─ agent-common (헬스)
              └─ agent-core (운영팀)
```

| 종류 | 개수 | 의미 |
|---|---|---|
| Primary | **항상 1개** | "본인 호수" — 새 파일 만들면 이 모임 소유가 됨 |
| Supplementary | **0개 이상** | "추가 가입 모임" — 그 모임 자료실 들어갈 권한 |

### 2. 정보가 어디 적혀 있나

```
/etc/passwd   주민 명단 (이름, 호수번호, 거주 위치, 사용 셸)
/etc/group    동호회 명단 (모임 이름, 멤버들)
/etc/shadow   비밀번호 (관리자만 볼 수 있음)
```

`/etc/passwd` 한 줄을 풀어보면:
```
agent-admin:x:1001:1001:Agent Admin:/home/agent-admin:/bin/bash
└────┬────┘ │ │    │   └─────┬───┘ └────────┬────────┘ └────┬───┘
   주민이름   │ 사번 │       이름표        본인 집           기본 셸
            │     본인 호수 (primary)
            비밀번호 (실제는 shadow에 따로)
```

## 한 번 보자 (Mac에서도 거의 됨)

```bash
id                          # 본인 정보
id agent-admin              # 특정 주민 정보
groups agent-admin          # 가입한 모임들
cat /etc/passwd | head      # 주민 명단 일부 (Mac도 비슷한 파일)
cat /etc/group | head       # 모임 명단 일부
```

## 주민·모임 만들기 (B1-1에서 필요)

```bash
# 모임 만들기
sudo groupadd agent-common
sudo groupadd agent-core

# 주민 등록 (본인 집 + 기본 셸 지정)
sudo useradd -m -s /bin/bash agent-admin
# -m: 본인 집(home) 만들기
# -s: 사용할 셸

# 주민을 모임에 가입시키기
sudo usermod -aG agent-common agent-admin
sudo usermod -aG agent-core agent-admin
# -aG: "기존 모임 유지하고 추가"
# ★ -a 빼면 다 날림 — 흔한 실수!
```

## 자주 헷갈리는 것

- **`usermod -G` 만 써도 추가될 줄 안다** → ❌ **기존 모임 다 날아감**. 반드시 `-aG`.
- **모임 가입 직후 바로 적용** → ❌ 새 셸/SSH 다시 접속해야 적용.
- **root와 sudo는 같은 것** → 비슷하지만 다름. sudo는 "잠깐 root처럼 행동" + 기록 남음.

## 이번 과제 매핑

```
주민          본인 호수        가입 모임
agent-admin   agent-admin      agent-common, agent-core
agent-dev     agent-dev        agent-common, agent-core
agent-test    agent-test       agent-common
```

**왜 이렇게?**
- 셋 다 `agent-common` → 공유 폴더(`upload_files`)는 셋 다 접근
- `agent-test`는 `agent-core` 빠짐 → 중요 폴더(`api_keys`, 로그) 못 봄

## 기술 용어 풀이

- **UID (User ID)** — 주민 사번 (정수, 1001, 1002, …). root는 UID 0.
- **GID (Group ID)** — 모임 번호.
- **Primary group** — 본인 호수 (한 명당 1개).
- **Supplementary group** — 추가 가입 모임 (여러 개 가능).
- **shadow** — 비밀번호 해시만 따로 저장하는 파일 (보안용).

## 더 알아보고 싶으면
- 명령: `man useradd`, `man usermod`, `man groups`
- 다음 노트: [file-permissions.md](./file-permissions.md) — 방마다 누가 들어올 수 있는지

## 출처
- B1-1 (Layer 1.2)
- 학습일: 2026-05-07
