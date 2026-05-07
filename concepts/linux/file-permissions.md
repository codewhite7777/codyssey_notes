# 파일 권한 (rwx)

## 한 줄 정의
파일과 디렉토리에 **3그룹(owner / group / other) × 3비트(r/w/x) = 9비트** 권한을 부여해 접근을 통제하는 리눅스 기본 보안 메커니즘.

## 왜 필요한가
- 시스템 파일(`/etc/shadow`)을 일반 사용자가 못 읽게
- 내 파일을 남이 못 수정하게
- 실행 가능한 스크립트와 그렇지 않은 데이터 파일 구분
- → 다중 사용자 시스템의 가장 기초적인 보안

## 핵심 원리

### 9비트 구조
```
-rwxr-x---  1 agent-dev agent-core  monitor.sh
│└─┬─┘└┬┘└┬┘    └────┬────┘  └────┬────┘
│  │   │  │         소유자       그룹
│  │   │  └── other (외부인) 권한
│  │   └────── group 권한
│  └────────── owner (소유자) 권한
└───────────── 파일 타입 (- 일반파일, d 디렉토리, l 심볼릭링크 등)
```

### r/w/x의 의미

| 비트 | 파일에서 | 디렉토리에서 |
|---|---|---|
| `r` (read) | 내용 읽기 | 안의 항목 **목록** 보기 (`ls`) |
| `w` (write) | 내용 수정 | 안에서 **파일 생성/삭제/이름변경** |
| `x` (execute) | **실행** 가능 | 안으로 **들어가기** (`cd`), 안의 파일 접근 |

> ★ 디렉토리에서 `x` 없으면 **`ls`로 이름 봐도 그 안 파일에 접근 못함**. 이건 자주 헷갈림.

### 8진법 표기
각 그룹의 rwx를 3비트 2진수 → 8진법 1자리:

```
r=4, w=2, x=1 (합산)

7 = rwx (4+2+1)
6 = rw- (4+2)
5 = r-x (4+1)
4 = r-- (4)
3 = -wx (2+1)
2 = -w- (2)
1 = --x (1)
0 = --- (0)
```

`chmod 750 file`:
- 7 (owner) = rwx — 만든 사람: 다 가능
- 5 (group) = r-x — 같은 그룹: 읽기·실행 (수정 X)
- 0 (other) = --- — 외부인: 아무것도 못 함

### 소유권 변경
```bash
chown <user>:<group> file
chown agent-dev:agent-core monitor.sh

chown agent-dev file          # 그룹 변경 X
chown :agent-core file        # 사용자 변경 X
```

### umask: 새 파일의 기본 권한
- 새 파일 만들 때 적용되는 "삭제할 권한" 마스크
- `umask 022` → 새 파일은 `666 - 022 = 644` (rw-r--r--), 새 디렉토리는 `777 - 022 = 755` (rwxr-xr-x)
- `umask 077` → 본인 외 아무도 못 봄 (보안 민감 환경)
- 셸 init 파일(`.bashrc` 등)에서 설정

## 자주 쓰는 명령

```bash
ls -l file                  # 권한 확인
ls -la dir/                 # 숨김 파일 포함 + 권한
chmod 750 monitor.sh        # 8진법으로 통째로
chmod u+x file              # 심볼릭: owner에 x 추가
chmod g-w file              # group에서 w 제거
chmod o= file               # other 권한 0 (아무것도 X)
chmod -R 750 dir/           # 재귀 (디렉토리 + 안의 모든 파일)
chown agent-dev:agent-core file
umask                       # 현재 umask 보기
umask 077                   # umask 변경 (현재 셸만)
```

## 흔한 오해

- ❌ "디렉토리에 r 권한 있으면 안의 파일 읽을 수 있다" → ✅ r은 **목록 보기**일 뿐. 안의 파일 접근하려면 디렉토리에 **x** 필요.
- ❌ "chmod 777 = 모두에게 모든 권한 = 편함" → ✅ **보안 재앙**. 누구나 그 파일·디렉토리를 수정·삭제 가능.
- ❌ "내가 만든 파일이니까 항상 수정 가능" → ✅ owner라도 `chmod`로 자기 권한 빼면 못함 (root만 우회).
- ❌ "심볼릭 링크 권한이 중요" → ✅ 심볼릭 링크 자체 권한은 보통 무의미. **가리키는 대상**의 권한이 적용.
- ❌ "디렉토리 -R 권한 변경해도 새 파일은 그대로" → ✅ `-R`은 **현재 존재하는 것**만. 새로 만든 파일은 **umask** 적용.

## 이번 과제(B1-1)에서

```
monitor.sh 권한 = 750 (rwxr-x---)
   owner = agent-dev (7=rwx) — 개발자가 수정·실행
   group = agent-core (5=r-x) — admin이 cron으로 실행 (admin이 core에 속하니까)
   other = (0=---)            — agent-test 등은 못 봄

api_keys/ = group agent-core만 접근
   → other에 권한 0 + 디렉토리 g+rwx 필요
   → 단순 chmod로는 "한 그룹만"이 어려움 → ACL 필요할 수도

upload_files/ = group agent-common 읽기·쓰기
   → 디렉토리 권한 g+rwx
```

> 단, "한 디렉토리에 group A는 RW, group B는 R" 같은 요구는 기본 권한으로 표현 불가 (group은 1개뿐). → 다음 노트 [acl.md] (Layer 2에서 학습)

## 관련 개념
- [users-and-groups.md](./users-and-groups.md)
- (예정) acl.md — 기본 권한의 한계 돌파
- [shell-environment.md](./shell-environment.md) — umask는 셸 init에서 설정

## 출처
- B1-1 (Layer 1.3)
- 학습일: 2026-05-07
- 참고: `man chmod`, `man umask`, `man chown`
