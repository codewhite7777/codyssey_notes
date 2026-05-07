# POSIX 파일 권한 모델

## 개요
모든 파일·디렉토리에 12비트 권한 메타데이터(9비트 rwx + 3비트 setuid/setgid/sticky)와 owner uid·group gid가 부여된다. 커널은 `open()`, `execve()` 등 syscall 처리 시 호출자의 신원과 이 메타데이터를 비교해 접근 허용 여부를 결정한다.

## 왜 알아야 하나
- syscall 레벨 보안의 1차 게이트 (DAC — Discretionary Access Control)
- 보안 사고의 다수가 잘못된 권한 (`chmod 777`, 잘못된 `umask`)
- ACL, MAC(SELinux/AppArmor), capabilities는 모두 이 위에 쌓임
- monitor.sh 같은 운영 스크립트의 권한 설계는 정확한 모델 이해 필수

## 12비트 mode

```
mode = 12비트
  ┌─ setuid    (4000)  실행 시 effective UID = file owner
  │  setgid    (2000)  파일: effective GID 변경 / 디렉토리: 새 파일이 디렉토리 group 상속
  │  sticky    (1000)  디렉토리: 자기 파일만 삭제 가능 (예: /tmp)
  ├─ owner rwx (0700)
  ├─ group rwx (0070)
  └─ other rwx (0007)
```

`ls -l`에서 특수 비트 표시:
```
-rwsr-xr-x   ← s (setuid + x), execute 가능
-rwSr--r--   ← S (setuid + no x), 의미 없음 (보통 잘못된 설정)
drwxr-x--T   ← T (sticky + no other-x), 잘 안 씀
drwxrwsrwx   ← s (setgid 디렉토리), 새 파일이 부모 group 상속
```

## r/w/x — 파일 vs 디렉토리

| | 파일 | 디렉토리 |
|---|---|---|
| `r` | 내용 read | dirent 목록 (`ls`) |
| `w` | 내용 modify | 디렉토리 내 파일 **생성·삭제·rename** |
| `x` | execve 가능 | path traversal — 안의 inode 접근 |

**디렉토리 `x` 없으면**: `ls`로 이름 봐도 안의 파일에 어떤 syscall도 통과 못 함 (`stat`, `open` 모두 EACCES).

**파일 삭제 권한**: 파일 자체의 w가 아니라 **부모 디렉토리의 w + x**. 그래서 read-only 파일도 부모에 권한 있으면 삭제 가능. sticky 비트로 방어.

## 권한 검사 알고리즘 (커널)

```
process가 file에 접근 시도
  ├─ root(uid=0)? → 항상 허용 (CAP_DAC_OVERRIDE 가진 경우)
  ├─ process.uid == file.owner_uid?
  │     → owner 권한 비트 검사
  ├─ process.gid == file.group_gid OR file.group_gid in process.supplementary[]?
  │     → group 권한 비트 검사
  └─ 그 외 → other 권한 비트 검사
```

★ owner > group > other **순서대로 매칭되는 첫 번째**가 적용. owner 권한이 없으면 group이 있어도 거부됨 (즉, owner의 권한을 더 좁게 둘 수 있음).

★ 경로 resolution은 각 디렉토리에 대해 **traverse(x) 권한** 필요. `/a/b/c`에서 `/a`나 `/a/b`에 x가 없으면 `/a/b/c`에 다른 권한이 충분해도 접근 불가.

## 8진법 표기

```
r = 4, w = 2, x = 1
─────────────────────
7 = rwx     6 = rw-     5 = r-x     4 = r--
3 = -wx     2 = -w-     1 = --x     0 = ---
```

`chmod 4750 file`:
- `4` = setuid (특수 비트)
- `7` = owner rwx
- `5` = group r-x
- `0` = other ---

## umask: 새 파일의 기본 mode

`open(path, O_CREAT, mode)` 시 실제 mode = `mode & ~umask`.

```bash
umask 022                      # 기본값 (대부분 배포판)
# 새 파일: 0666 & ~022 = 0644 (rw-r--r--)
# 새 디렉토리: 0777 & ~022 = 0755 (rwxr-xr-x)

umask 077                      # 본인만 (보안 환경)
umask 027                      # 같은 그룹은 read 가능
```

★ umask는 셸별 상태 + 자식 프로세스에 상속. `.bashrc`나 `/etc/login.defs`로 영구 설정.

## setgid 디렉토리 — 협업 폴더의 핵심 패턴

```bash
chmod g+s /shared/dir          # setgid bit set
ls -ld /shared/dir             # drwxrws---

# 효과: 이 디렉토리에 만들어지는 모든 파일은 부모 디렉토리의 group을 상속
#       → 사용자별 primary group이 달라도 공유 그룹 유지
```

이게 없으면 `agent-admin`과 `agent-dev`가 같이 쓰는 폴더에서 파일마다 group owner가 제각각 됨.

## 한 번 보자

```bash
ls -l file                     # 권한 + owner + group
ls -ln file                    # uid/gid를 숫자로
stat file                      # 더 자세 (Mode: 비트별 분해)
namei -l /path/to/file         # 경로 따라 각 디렉토리 권한

chmod 750 monitor.sh           # 8진법
chmod u=rwx,g=rx,o= monitor.sh # 심볼릭
chmod -R u+rX dir/             # X = "디렉토리이거나 이미 x인 파일에만 x" (안전한 재귀)
chmod g+s dir/                 # setgid 디렉토리

chown agent-dev:agent-core monitor.sh
chgrp -R agent-core dir/

umask                          # 현재 (네 자리)
umask -S                       # 심볼릭 표시

# DAC 검증
sudo -u agent-test cat /var/log/agent-app/monitor.log  # EACCES 기대
sudo -u agent-test stat /var/log/agent-app/            # 디렉토리 권한 검증
```

## 흔한 함정

- **`chmod 777` = 보안 재앙** — write 권한이 모두에게 = 임의 백도어 설치 가능. 디렉토리는 임의 파일 생성·삭제.
- **재귀 chmod에 숫자 사용** — `chmod -R 755`는 데이터 파일에도 x를 주어 위험. 대신 `chmod -R u+rwX,g+rX,o-rwx` (대문자 X 활용).
- **삭제 권한 = 부모 디렉토리 권한** — read-only 파일도 부모 w 있으면 삭제됨. sticky 비트로 방어 (`/tmp` 모델).
- **setuid 셸 스크립트** — 거의 모든 OS가 무시 (race condition). 진짜 필요하면 컴파일된 wrapper.
- **owner가 group보다 권한 적으면** — 매칭되는 첫 단계가 owner이므로 group 권한 못 받음. 직관과 반대.
- **NFS의 권한** — 클라이언트의 uid/gid가 서버에서 다른 사용자를 가리킬 수 있음 (`no_root_squash` 등 옵션 주의).
- **umask는 프로세스 상속 속성** — 자식 프로세스에게 전파. cron 환경의 umask는 기본 022 (cron 데몬에서 상속).
- **심볼릭 링크는 권한 무관** — 링크 자체의 권한은 보통 `lrwxrwxrwx`로 항상 동일. `chmod`는 대상에 적용 (`-h` 옵션 예외).

## B1-1 매핑

```
monitor.sh  =  -rwxr-x---  (0750)  agent-dev:agent-core
   owner(agent-dev)  rwx  개발자가 수정·실행
   group(agent-core) r-x  agent-admin이 cron으로 실행 (core 멤버)
   other             ---  agent-test 차단

api_keys/   =  drwxr-x---  (0750)  agent-admin:agent-core
   group(agent-core) r-x  core 멤버만 traverse + 목록
   안의 t_secret.key는 -r--r-----  (0440)  agent-admin:agent-core
   → core 멤버 read-only

/var/log/agent-app/  =  drwxrwx---  (0770) + setgid?
   group(agent-core) rwx  core 멤버가 로그 파일 생성·rotation
   setgid 추천 → 새 로그 파일도 agent-core 그룹 상속
```

> "한 디렉토리에 group A는 RW, group B는 R" 같은 요구는 9비트로 표현 불가 → ACL 필요 (Layer 2 학습 토픽).

## 인접 토픽

- **POSIX ACL** — owner/group/other 모델의 한계 돌파 (`setfacl`, `getfacl`)
- **MAC (SELinux, AppArmor)** — DAC 위에 추가되는 강제 정책 layer
- **capabilities** — root의 모든 능력을 39+개 단위로 분리 (`cap_net_bind_service` 등)
- **immutable bit** (`chattr +i`) — root조차 수정 못 함
- **mount option** — `noexec`, `nosuid`, `nodev`로 파티션 단위 권한 제한
- **fanotify, audit subsystem** — 권한 위반 추적

## 참고
- `man 1 chmod`, `man 2 open`, `man 7 path_resolution`
- `man 7 inode` — mode 비트 정의
- `man 2 access`, `man 2 faccessat` — 권한 검사 syscall

---
출처: B1-1 (Layer 1.3) · 학습일: 2026-05-07
