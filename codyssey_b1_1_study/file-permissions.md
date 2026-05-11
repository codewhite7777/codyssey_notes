# POSIX 파일 권한 모델

## 개요

Linux는 모든 파일과 디렉토리에 권한 메타데이터를 부여하고, 시스템 콜(`open()`, `execve()`, `unlink()` 등)이 발생할 때마다 호출 프로세스의 신원과 이 메타데이터를 비교해 접근 허용 여부를 결정한다. 이 권한 시스템은 **DAC (Discretionary Access Control)**라 불리는데, "discretionary"라는 단어가 의미하듯 권한 부여의 결정권이 파일 소유자에게 있다는 뜻이다 — 강제적 정책(MAC: SELinux, AppArmor)과 대비된다.

POSIX 권한 모델은 1970년대 UNIX 설계의 산물로, 단순함과 표현력의 절묘한 균형을 이룬다. 단 12비트로 owner/group/other 세 가지 주체와 read/write/execute 세 가지 동작, 그리고 setuid/setgid/sticky 세 가지 특수 동작을 모두 표현한다. 이 단순함이 50년이 지난 지금까지 거의 변하지 않고 사용되는 이유다. 이후의 ACL, capabilities, MAC 정책은 모두 이 12비트 모델을 보완하는 방향으로 추가되었지 대체하지 않았다.

## 왜 알아야 하나

파일 권한은 syscall 레벨 보안의 1차 방어선이다. SQL 인젝션이나 XSS 같은 application-layer 공격이 흔히 알려진 취약점이지만, 통계적으로 더 흔한 사고는 **권한 설정 실수**다. `chmod 777`로 임시로 풀어둔 디렉토리, root로 실행되는 cron 작업이 만든 자산을 일반 사용자가 수정 가능, 잘못된 umask로 새 파일이 모두에게 read 가능 — 이런 실수들은 코드 리뷰에서 잘 잡히지 않지만 production에서 데이터 유출의 원인이 된다.

운영자 입장에서도 권한 모델의 정확한 이해는 일상적으로 필요하다. monitor.sh 같은 운영 스크립트를 만들 때 누가 owner여야 하는지, 어떤 그룹이 실행 권한을 가져야 하는지, 새 로그 파일이 어떤 권한으로 생성되어야 하는지를 결정해야 한다. 이 결정들은 모두 12비트 모델의 정확한 매핑이며, 잘못 설정하면 cron이 실패하거나 의도하지 않은 사용자가 자격 증명에 접근하는 사고로 이어진다. 이번 과제도 권한 설계를 명세에서 직접 요구한다 — monitor.sh의 권한이 750이어야 하고 owner는 agent-dev여야 하며 group은 agent-core여야 하는 이유를 설명할 수 있어야 한다.

## 12비트 mode

파일의 권한 메타데이터는 정확히 12비트로 인코딩된다. 위쪽 3비트는 특수 동작(setuid, setgid, sticky), 아래쪽 9비트는 owner/group/other × r/w/x 매트릭스다.

```
mode = 12비트
  ┌─ setuid    (4000)  실행 시 effective UID가 file owner로 변경
  │  setgid    (2000)  파일: effective GID 변경 / 디렉토리: 새 파일이 디렉토리 group 상속
  │  sticky    (1000)  디렉토리: 자기 소유 파일만 삭제 가능 (예: /tmp)
  ├─ owner rwx (0700)
  ├─ group rwx (0070)
  └─ other rwx (0007)
```

`ls -l` 출력에서 특수 비트는 권한 문자에 합쳐져서 표현된다. setuid가 set이고 owner의 x도 있으면 `s`로 표시된다. setuid는 set이지만 x는 없으면 `S`로 표시되는데, 이는 보통 잘못된 설정(setuid가 실행 가능하지 않은 파일에 의미가 없으므로)을 의미한다.

```
-rwsr-xr-x   ← s (setuid + x), execute 가능 — passwd 같은 정상 케이스
-rwSr--r--   ← S (setuid + no x), 의미 없음 (보통 실수)
drwxr-x--T   ← T (sticky + no other-x), 잘 안 씀
drwxrwsrwx   ← s (setgid 디렉토리), 새 파일이 부모 group 상속 — 협업 폴더 패턴
```

## r/w/x — 파일과 디렉토리에서 의미가 다르다

read/write/execute 비트는 파일과 디렉토리에서 미묘하게 다른 의미를 가진다. 이 차이를 정확히 이해하지 못하면 권한을 설정해도 의도한 효과가 안 나오거나, 의도하지 않은 우회 경로가 생긴다.

| | 파일 | 디렉토리 |
|---|---|---|
| `r` | 내용 read | dirent 목록 (`ls`) |
| `w` | 내용 modify | 디렉토리 내 파일 **생성·삭제·rename** |
| `x` | execve 가능 | path traversal — 안의 inode 접근 |

가장 자주 헷갈리는 부분은 디렉토리에서 `x`의 의미다. `x` 비트는 "안으로 들어가는 권한"이며, 더 정확히는 **path resolution을 통과할 수 있는 권한**이다. 디렉토리에 `r`만 있고 `x`가 없으면 `ls`로 안의 파일 이름은 볼 수 있지만, 그 파일에 대한 어떤 syscall(`stat`, `open` 등)도 EACCES로 실패한다. 반대로 `x`만 있고 `r`이 없으면 `ls`는 실패하지만, 정확한 파일 이름을 알면 그 파일에 접근할 수 있다.

또 다른 자주 놓치는 부분은 **파일 삭제 권한이 파일 자체가 아닌 부모 디렉토리에 있다**는 사실이다. `unlink()` syscall은 디렉토리 엔트리를 제거하는 동작이므로, 부모 디렉토리에 `w`와 `x` 권한만 있으면 그 안의 read-only 파일도 삭제할 수 있다. 이 동작이 보안 문제를 일으킬 수 있어서, `/tmp`처럼 모두가 쓸 수 있는 디렉토리에는 sticky 비트를 set해서 "자기 소유 파일만 삭제 가능"이라는 추가 제약을 건다.

## 권한 검사 알고리즘

커널이 파일 접근 권한을 검사하는 알고리즘은 다음과 같다.

```
process가 file에 접근 시도
  ├─ root(uid=0)이고 CAP_DAC_OVERRIDE 가짐? → 항상 허용
  ├─ process.uid == file.owner_uid?
  │     → owner 권한 비트 검사 (owner 비트가 안 맞으면 거부, group 검사 안 함)
  ├─ process.gid == file.group_gid OR file.group_gid in process.supplementary[]?
  │     → group 권한 비트 검사
  └─ 그 외 → other 권한 비트 검사
```

이 알고리즘에서 **owner > group > other 순서로 매칭되는 첫 번째만 적용된다**는 점이 직관과 다른 부분이다. 즉, owner 권한을 더 좁게 두면 group 권한이 있어도 owner는 그 권한을 받지 못한다. 예를 들어 `chmod 070 file`처럼 owner만 권한 0이고 group은 rwx면, 파일 owner인 사용자는 (owner 매칭이 먼저 되므로) read조차 못 한다. 이런 설정은 의도적으로는 거의 쓰이지 않지만, 정책 자동화에서 사고로 만들 수 있다.

또 한 가지 중요한 점은 **path resolution도 권한 검사 대상**이라는 사실이다. `/a/b/c/file`에 접근하려면 호출자가 `/`, `/a`, `/a/b`, `/a/b/c` 모든 디렉토리에 traverse 권한(`x`)을 가져야 한다. 중간 디렉토리 하나라도 권한이 없으면 최종 파일에 어떤 권한이 있어도 접근 불가다. 이 사실은 협업 디렉토리 설계 시 자주 놓치는 부분이다.

## 8진법 표기와 chmod

권한을 명령으로 설정할 때 가장 자주 쓰는 표기법은 8진법이다. 각 r/w/x를 4/2/1로 매핑하고 합산한다.

```
r = 4, w = 2, x = 1
─────────────────────
7 = rwx     6 = rw-     5 = r-x     4 = r--
3 = -wx     2 = -w-     1 = --x     0 = ---
```

특수 비트가 있으면 4자리 숫자가 된다. `chmod 4750 file`은 다음과 같이 읽는다 — 첫 자리 4는 setuid, 7은 owner rwx, 5는 group r-x, 0은 other 0. 이 패턴은 sudo·passwd 같은 setuid 바이너리에 흔히 보인다.

chmod는 8진법 외에 심볼릭 표기도 지원한다. `chmod u+x file`은 owner에 execute를 추가하고, `chmod g-w,o-rwx file`은 group에서 write를 빼고 other를 모두 차단한다. 재귀 변경(`-R`) 시에는 심볼릭 표기에 대문자 X를 쓰는 것이 안전하다 — 대문자 X는 "디렉토리이거나 이미 어떤 카테고리에서 x를 가진 파일에만" execute를 부여하므로, 데이터 파일에 실수로 x를 주는 일을 방지한다. `chmod -R u+rwX,g+rX,o-rwx dir/`처럼 사용한다.

## umask: 새 파일의 기본 mode

새 파일을 만들 때마다 권한을 명시적으로 지정할 필요는 없다. 모든 프로세스는 **umask**라는 마스크를 들고 있어서, 새 파일·디렉토리의 기본 mode를 자동으로 결정한다. 정확한 공식은 `실제 mode = 요청 mode & ~umask`다.

대부분의 배포판은 기본 umask로 022를 사용한다. 그 결과, 새 파일은 0644(rw-r--r--), 새 디렉토리는 0755(rwxr-xr-x)로 생성된다. 이는 "owner는 다 가능, 다른 사람은 read만"이라는 합리적 기본값이다. 보안이 강한 환경에서는 umask 077을 쓰면 새 파일이 0600(rw-------)으로 생성되어 owner 외에는 아무도 접근할 수 없다.

umask는 셸별 상태이며 자식 프로세스에 상속된다. 셸 시작 파일(`.bashrc`, `/etc/login.defs`)에서 영구 설정할 수 있다. 한 가지 주의할 점은 cron의 umask인데, 기본 022가 적용되므로 cron이 만드는 파일도 group/other에게 read 가능해진다. 보안 민감한 데이터를 다루는 cron 작업이라면 스크립트 첫 줄에서 명시적으로 `umask 077`을 설정해야 한다.

## setgid 디렉토리: 협업 폴더의 핵심 패턴

여러 사용자가 함께 쓰는 디렉토리에서 자주 만나는 문제는 **새 파일의 group owner가 만든 사람의 primary group이 되어버린다**는 점이다. agent-admin과 agent-dev가 같이 쓰는 폴더에서 각자 파일을 만들면 group owner가 제각각(agent-admin / agent-dev)이 되어, 다른 사람이 그 파일을 수정 못 하는 사고가 생긴다.

이 문제의 표준 해결책이 **setgid 디렉토리**다. 디렉토리에 setgid 비트를 set하면(`chmod g+s dir/`), 그 안에 만들어지는 모든 파일은 부모 디렉토리의 group을 자동으로 상속한다.

```bash
chmod g+s /shared/dir          # setgid bit set
ls -ld /shared/dir             # drwxrws---  (s가 setgid 표시)

# 이후 /shared/dir 안에 만든 파일은 누가 만들든 group owner가 디렉토리 group과 같음
```

이 패턴은 협업 디렉토리, 공유 로그 디렉토리, 빌드 산출물 디렉토리 등에 거의 필수적으로 사용된다. 이번 과제의 `/var/log/agent-app/`도 setgid를 설정하면 누가 로그 파일을 생성하더라도 group이 `agent-core`로 일관되게 유지된다.

## 한 번 보자

권한 관련 명령들을 직접 실행해보면 모델이 더 명확해진다. 먼저 권한을 확인하는 도구들이다.

```bash
ls -l file                     # 권한 + owner + group
ls -ln file                    # uid/gid를 숫자로 표시
stat file                      # 더 자세한 메타데이터 (Mode 비트별 분해 포함)
namei -l /path/to/file         # 경로의 각 디렉토리 권한을 따라가며 표시
```

특히 `namei -l`은 path resolution 권한을 디버깅할 때 유용하다 — 어느 중간 디렉토리에서 traverse가 막히는지 한눈에 보인다.

권한을 변경하는 명령들은 다음과 같다.

```bash
chmod 750 monitor.sh           # 8진법
chmod u=rwx,g=rx,o= monitor.sh # 심볼릭 (위와 동치)
chmod -R u+rwX,g+rX,o-rwx dir/ # 안전한 재귀 (대문자 X 활용)
chmod g+s dir/                 # setgid 디렉토리

chown agent-dev:agent-core monitor.sh  # owner와 group 동시 변경
chgrp -R agent-core dir/               # group만 변경 (재귀)

umask                          # 현재 umask (네 자리)
umask -S                       # 심볼릭 표시
umask 077                      # 보안 강한 환경
```

마지막으로 권한이 의도대로 작동하는지 검증할 때는 `sudo -u`로 다른 사용자를 시뮬레이션한다. 이번 과제의 권한 설계가 의도대로 작동하는지 확인할 때 유용하다.

```bash
sudo -u agent-test cat /var/log/agent-app/monitor.log  # EACCES 기대 (test는 core 아님)
sudo -u agent-test stat /var/log/agent-app/            # 디렉토리 권한 검증
sudo -u agent-admin /home/agent-admin/agent-app/bin/monitor.sh  # admin이 실행 가능한지
```

## 흔한 함정

권한 모델은 단순해 보이지만 운영에서 부딪히는 함정은 대부분 "당연하게 보이는 동작이 실제로는 다르게 작동하는" 종류다. 가장 흔한 첫 사고는 `chmod 777`을 권한 문제 해결책으로 쓰는 것이다. 임시로라도 777로 풀어두면 어떤 사용자든 그 파일·디렉토리를 수정·삭제할 수 있게 되고, 디렉토리에 777을 주면 누구나 임의 파일을 생성할 수 있어 백도어 설치 경로가 된다. "권한 문제는 정확한 권한 설정으로만 해결한다"가 원칙이며, 빠른 우회로 777을 쓰는 습관이 production에서 보안 사고로 이어진다.

비슷한 맥락에서 재귀 chmod에 숫자를 사용하는 것도 흔한 실수다. `chmod -R 755 dir/`을 실행하면 디렉토리뿐 아니라 그 안의 데이터 파일에도 x 비트가 부여되는데, 데이터 파일에 x는 의미 없을 뿐 아니라 그 파일이 우연히 ELF 헤더로 시작하면 실행 가능 바이너리로 인식되어 보안 도구가 경고를 내거나 의도하지 않은 동작을 할 수 있다. 재귀 변경에는 심볼릭 표기와 대문자 X를 사용해 "이미 디렉토리이거나 어디선가 x를 가진 파일에만" execute를 부여해야 안전하다.

권한 모델 자체의 메커니즘에서 비롯되는 함정도 있다. 대표적인 것이 파일 삭제 권한이 파일이 아니라 부모 디렉토리에 있다는 사실로, read-only로 보이는 파일이 사용자에 의해 삭제 가능한 이유가 부모 디렉토리에 write 권한이 있기 때문이다. sticky 비트(`chmod +t dir/`)로 "자기 소유 파일만 삭제 가능" 제약을 추가할 수 있지만, 이 메커니즘 자체를 모르면 의도하지 않은 삭제가 일어난다. 또 다른 알고리즘 함정으로, owner의 권한을 group보다 좁게 두면 owner는 그 권한을 받지 못한다는 점이 있다 — 권한 검사가 owner 매칭을 먼저 시도하고 매칭된 결과를 그대로 적용하기 때문이다. 의도적으로 이런 설정을 하는 일은 드물지만, 자동화 스크립트가 권한을 잘못 계산하면 사고로 만들 수 있다.

보안 가정 자체를 깨는 함정으로 setuid 셸 스크립트가 있다. 거의 모든 OS가 셸 스크립트에 대한 setuid를 무시하는데, 실행 시점에 race condition으로 다른 스크립트로 바꿔치기될 수 있는 보안 이슈 때문이다. setuid가 정말 필요하면 컴파일된 wrapper를 만들고 그 안에서 스크립트를 호출하거나, 더 일반적으로는 sudo 정책으로 대체한다.

분산·운영 자동화 환경에서는 또 다른 종류의 함정이 등장한다. NFS 환경에서는 클라이언트와 서버가 다른 사용자 매핑을 가지면 같은 uid가 다른 사용자를 가리킬 수 있고, `no_root_squash` 옵션을 켠 채 NFS export하면 클라이언트의 root가 서버에서도 root로 작용해 보안 모델이 깨진다. cron 환경에서는 umask 함정이 자주 만나는데, cron이 만드는 파일은 cron 데몬에서 상속한 umask 022로 생성되어 group/other에게 read 가능해진다. 보안 민감 데이터를 만드는 cron 스크립트는 첫 줄에서 명시적으로 umask를 강하게 설정해야 한다.

마지막으로 심볼릭 링크의 권한은 보통 무시된다. `ls -l symlink`로 보면 항상 `lrwxrwxrwx`로 보이지만 이 권한은 의미 없으며, 실제 권한 검사는 링크가 가리키는 대상에 적용된다. `chmod`로 심볼릭 링크의 권한을 바꾸려고 하면 대상이 바뀌고 링크 자체는 그대로 유지된다(`-h` 옵션은 예외).

## B1-1 매핑

이번 과제의 권한 설계를 위 모델로 분석해보자.

```
monitor.sh  =  -rwxr-x---  (0750)  agent-dev:agent-core
   owner(agent-dev)  rwx  개발자가 수정·실행
   group(agent-core) r-x  agent-admin이 cron으로 실행 (admin도 core 멤버)
   other             ---  agent-test 차단

api_keys/   =  drwxr-x---  (0750)  agent-admin:agent-core
   group(agent-core) r-x  core 멤버만 traverse + 목록
   안의 t_secret.key는 -r--r-----  (0440)  agent-admin:agent-core
   → core 멤버 read-only (수정 차단)

/var/log/agent-app/  =  drwxrwx---  (0770) + setgid
   group(agent-core) rwx  core 멤버가 로그 파일 생성·rotation
   setgid 추천 → 새 로그 파일도 agent-core 그룹 상속 (group 일관성 유지)
```

각 권한 결정의 합리성을 짚어보면 다음과 같다. monitor.sh의 owner가 agent-dev인 것은 "개발자가 작성·수정하는 자산"이라는 명세 의도를 반영한다. group이 agent-core인 것은 "운영자(agent-admin)가 cron으로 실행해야 한다"는 요구를 충족한다 — agent-admin이 agent-core 멤버이므로 group 권한 r-x로 실행 가능하다. other가 0인 것은 agent-test를 비롯한 외부인 차단이다.

api_keys 안의 키 파일이 0440인 것은 의도적으로 매우 좁다. 자격 증명은 read-only로 충분하고, 실수로 수정·삭제되면 안 되므로 owner의 write도 빼는 것이 표준이다. 이 파일을 갱신하려면 권한을 일시적으로 풀어야 하는데, 그 자체가 "이 작업은 의도적이고 신중해야 한다"는 신호 역할을 한다.

/var/log/agent-app/에 setgid를 추가 권장하는 이유는 앞서 설명한 협업 폴더 패턴이다. monitor.sh가 agent-admin으로 실행되어 새 로그 파일을 만들면, setgid 없이는 그 파일의 group owner가 agent-admin이 되어 다른 core 멤버가 수정 못 하는 문제가 생길 수 있다. setgid를 켜면 새 파일도 agent-core 그룹으로 통일된다.

한 가지 흥미로운 점은 이 모든 권한 요구가 9비트 모델로 표현 가능하다는 사실이다. 하지만 만약 명세가 "한 디렉토리에 group A는 RW, group B는 R만"을 요구했다면 9비트로는 표현 불가능하다 — group은 한 번에 하나만 지정할 수 있기 때문이다. 그럴 때 필요한 것이 ACL이며, Layer 2에서 자세히 다룬다.

## 인접 토픽

권한 모델의 확장 학습은 DAC 모델의 한계를 보완하는 세 갈래 — 확장(ACL), 강제(MAC), 분리(capabilities) — 로 정리해 볼 수 있다.

기본 9비트로 표현할 수 없는 권한 요구를 다루는 확장이 POSIX ACL이다. `setfacl`, `getfacl` 명령으로 owner/group/other를 넘어 임의의 사용자·그룹별 세밀한 권한을 부여할 수 있고, ext4·XFS 같은 대부분의 현대 파일시스템이 지원한다. "한 디렉토리에 group A는 RW, group B는 R" 같은 요구가 정확히 ACL의 영역이며, 다음 노트에서 자세히 다룬다.

DAC의 근본적 한계 — root가 모든 정책을 우회할 수 있다는 점 — 을 보완하는 layer가 MAC (Mandatory Access Control)이다. SELinux나 AppArmor로 구현되며, DAC 위에 추가되는 강제 정책으로 root조차 우회할 수 없게 만든다. 컨테이너 보안, 데몬 격리, 시스템 강화(hardening) 작업의 핵심 도구다.

권한을 더 잘게 쪼개는 방향에서는 capabilities가 있다. 전통적으로 "root이거나 root가 아니거나"의 이분법이었던 권한을 39개 이상의 단위로 분리한 메커니즘으로, 예를 들어 ping이 root setuid 없이 raw 소켓을 사용할 수 있도록 `cap_net_raw`만 부여하는 식이다. setuid의 안전한 대안으로 권장된다.

이 외에도 운영 환경에서 알아두면 좋은 두 가지 보조 메커니즘이 있다. immutable 비트(`chattr +i file`)는 root조차 수정·삭제할 수 없게 만드는 확장 속성으로 시스템 무결성 보호에 사용되며, 일반 chmod로는 보이지 않아 디버깅 시 함정이 되기도 한다. mount option은 파티션 단위 보안을 제공하는데, `noexec`로 그 파티션에서 실행 파일을 못 돌리게, `nosuid`로 setuid 비트를 무시하게, `nodev`로 디바이스 노드를 못 만들게 할 수 있어 일반적으로 `/tmp`나 외부 마운트에 적용한다.

## 참고

- `man 1 chmod` — chmod의 모든 옵션
- `man 2 open` — 파일 권한과 syscall의 관계
- `man 7 path_resolution` — 경로 권한 검사 알고리즘
- `man 7 inode` — mode 비트의 정식 정의
- `man 2 access`, `man 2 faccessat` — 권한 검사 syscall

---
출처: B1-1 (Layer 1.3) · 학습일: 2026-05-07
