# 사용자·그룹·신원 (Identity)

## 개요

Linux의 멀티유저 권한 모델은 모든 행위 — 파일 생성, 프로세스 실행, 네트워크 접속 — 가 특정 "신원"에 귀속된다는 전제 위에 만들어졌다. 이 신원은 사용자 식별자(UID)와 그룹 식별자(GID)의 조합으로 표현되며, 한 사용자는 여러 그룹에 속할 수 있다. 모든 파일과 프로세스는 자신을 만든 신원의 정보를 메타데이터로 들고 다니고, 커널은 시스템 콜이 발생할 때마다 이 메타데이터를 검사해 허용 여부를 결정한다.

이 모델은 1970년대 UNIX의 설계에서 비롯되었지만, 여전히 현대 컨테이너의 user namespace, Kubernetes의 ServiceAccount, sudo의 정책 엔진까지 모두 이 위에 쌓여 있다. 따라서 멀티유저 격리·권한 제어를 다루는 거의 모든 작업은 이 모델의 정확한 이해를 전제한다.

## 왜 알아야 하나

권한과 신원의 부정확한 운영은 보안 사고의 가장 흔한 원인 중 하나다. root 계정을 일상적으로 사용하거나, 여러 사람이 공유 계정을 쓰거나, supplementary group을 잘못 설정해 의도하지 않은 권한을 부여하거나, sudo 정책을 너무 느슨하게 잡으면 — 이런 실수들이 모여 privilege escalation이라 불리는 보안 위협의 토대가 된다.

운영 측면에서도 신원 모델의 이해는 매우 실용적이다. 협업 디렉토리를 만들 때 어떤 그룹을 부여해야 하는지, cron 작업을 누가 실행해야 안전한지, 컨테이너의 root가 호스트에서 어떤 의미인지 — 이런 결정은 모두 이 모델의 세부를 알아야 정확히 할 수 있다. 이번 과제도 정확히 이 영역을 다룬다. 운영(`agent-admin`), 개발(`agent-dev`), 테스트(`agent-test`)라는 역할별 사용자를 두고, 공유 그룹(`agent-common`)과 핵심 그룹(`agent-core`)으로 데이터 접근 영역을 분리하는 설계는 실무에서 흔히 보는 권한 구조의 단순화된 버전이다.

## 신원의 구성 요소

프로세스 하나가 들고 다니는 신원은 단순히 "uid 1001" 정도의 정보가 아니다. 커널 내부의 `cred` 구조체는 다음과 같이 여러 종류의 식별자를 보관한다.

```
프로세스 신원 (커널의 cred struct)
   ├─ Real UID/GID         로그인한 실제 사용자
   ├─ Effective UID/GID    실제 권한 검사에 사용 (setuid 시 변경됨)
   ├─ Saved UID/GID        권한 임시 drop/restore에 보존
   ├─ Filesystem UID/GID   파일 접근 시 사용 (보통 effective와 같음, NFS legacy)
   └─ Supplementary GIDs[] 추가 그룹 집합 (NGROUPS_MAX 한계, 보통 65536)
```

세 가지 UID(real, effective, saved)가 분리되어 있는 이유는 특권 프로그램의 권한 위임 메커니즘 때문이다. `passwd` 명령을 일반 사용자가 실행하는 경우를 생각해보자. 커널은 그 프로세스의 effective UID를 root로 바꿔서 (setuid 비트의 효과) `/etc/shadow` 같은 root 전용 파일을 쓸 수 있게 만들지만, real UID는 여전히 호출자라서 "지금 누가 이 작업을 시켰는지"는 추적 가능하다. saved UID는 effective를 임시로 일반 사용자로 떨어뜨렸다가 다시 root로 복원할 때 사용된다. 이런 분리 구조는 sudo, su, screen lock, 일부 데몬의 권한 관리에 모두 활용되며, 단순해 보이는 "사용자 신원"이 실제로는 권한 동작을 정밀하게 통제하기 위한 다층 구조라는 점을 드러낸다.

## Primary vs Supplementary Group

한 사용자가 여러 그룹에 속할 수 있다는 사실에서 두 종류의 멤버십이 구분된다. 이 구분은 새 파일을 만들 때 어떤 그룹이 소유 그룹이 되는지를 결정하기 때문에 운영에서 매우 중요하다.

primary group은 사용자당 정확히 하나만 가질 수 있고, `/etc/passwd`의 4번째 필드(GID)에 적혀 있다. 이 그룹의 가장 중요한 역할은 그 사용자가 새 파일을 만들 때의 기본 group owner가 된다는 점이다 (단 setgid 디렉토리에서는 예외). 보통 사용자명과 같은 이름의 그룹이 useradd 시점에 자동으로 만들어져 primary로 지정된다. 반면 supplementary group은 0개 이상 자유롭게 가질 수 있고, `/etc/group`에서 그룹별 멤버 목록에 사용자명이 등록되는 형태로 표현된다. 이 그룹들은 새 파일 소유에는 영향을 주지 않지만 기존 파일에 접근할 때 권한 검사에 사용된다 — 커널은 파일의 group GID와 호출자의 모든 supplementary GID를 비교해서 매칭되면 group 권한을 적용한다.

```
사용자 agent-admin
  ├─ uid: 1001
  ├─ primary gid: 1001 (agent-admin)
  └─ supplementary: [1002 (agent-common), 1003 (agent-core)]
```

위 예에서 `agent-admin`이 만든 새 파일의 group은 기본적으로 `agent-admin`이 되지만, `/var/log/agent-app/` 같이 `agent-core` 소유 디렉토리에 들어가서 작업할 때는 supplementary 매칭으로 `agent-core` group의 권한이 적용된다. 이 구분이 중요한 또 다른 이유는 곧 다룰 `usermod -G` 명령의 함정과 직결되는데, 이 명령이 supplementary group을 통째로 교체하는 동작이라서 잘못 쓰면 사용자가 갑자기 모든 협업 그룹에서 추방되는 사고로 이어질 수 있기 때문이다.

## 정보 저장 위치와 NSS

전통적으로 사용자·그룹 정보는 다음 파일들에 저장된다.

```
/etc/passwd     사용자 (이름, uid, primary gid, GECOS, home, shell)
/etc/shadow     비밀번호 해시 + 만료 정책 (root만 read 가능, 0640 root:shadow)
/etc/group      그룹 (이름, gid, 멤버 목록)
/etc/gshadow    그룹 패스워드 (현대에는 거의 안 씀)
```

이 중 가장 자주 보는 `/etc/passwd`의 한 줄을 풀어보면 각 필드의 의미가 명확해진다.

```
agent-admin:x:1001:1001:Agent Admin:/home/agent-admin:/bin/bash
└──┬──────┘ │ └┬─┘ └┬─┘ └────┬───┘ └─────┬───────┘ └────┬───┘
   name     │ uid  pgid    GECOS         home         login shell
            password placeholder ('x' = use shadow)
```

GECOS 필드는 General Electric Comprehensive Operating System의 약자로, 1970년대 호환성에서 비롯된 역사적 잔재이지만 여전히 사용된다. 이 필드는 콤마(`,`)로 구분된 sub-field(전체 이름, 사무실, 직장 전화, 집 전화)를 가질 수 있어서 `chfn` 명령으로 변경 가능한데, 콤마가 들어간 값을 쓰려고 하면 파싱이 깨지는 함정이 있다.

현대 시스템에서는 이 파일들만이 신원 정보의 출처가 아니다. NSS (Name Service Switch)라는 추상화 layer가 `/etc/nsswitch.conf`에 정의된 우선순위에 따라 여러 백엔드 — 파일, LDAP, SSSD, systemd-homed 등 — 를 순차적으로 조회한다. 그래서 회사 환경에서 LDAP을 쓰면 `/etc/passwd`에 없는 사용자도 `getent passwd alice`로 조회된다. 이 사실이 `id` 명령이나 `getent` 같은 도구가 왜 `/etc/passwd`를 직접 cat한 것보다 더 많은 정보를 보여줄 수 있는지를 설명한다.

## 한 번 보자

지금까지 설명한 개념을 직접 확인해보자. 가장 자주 쓰는 명령은 `id`다.

```bash
id                                  # 현재 신원 (uid/gid/groups 한 번에)
id -G                               # supplementary GID만 (수치)
id -Gn                              # supplementary GID 이름
```

`id` 출력의 각 필드가 위 설명과 어떻게 매칭되는지 직접 보면 모델이 머리에 박힌다. 이어서 NSS를 통해 사용자·그룹을 조회한다.

```bash
getent passwd agent-admin           # NSS 통해 조회 (LDAP 등 백엔드 포함)
getent group agent-core             # 그룹 멤버 목록
```

그룹 멤버십을 변경하려면 다음 명령을 사용하는데, 한 가지 중요하게 기억할 점은 변경이 즉시 반영되지 않는다는 사실이다. 커널이 프로세스의 GID set을 `setgroups()` 호출 시점에만 업데이트하므로, 기존에 열려 있는 셸은 옛 GID를 계속 사용한다. 새 SSH 세션이나 `newgrp`로 새 셸을 띄워야 적용된다.

```bash
sudo usermod -aG agent-core agent-dev    # 추가 (★ -a 없으면 다 날림)
sudo gpasswd -a agent-dev agent-core     # 동일 동작 (gpasswd 사용)
newgrp agent-core                        # 새 셸에서 primary group 일시 변경
```

마지막으로 setuid의 실제 효과를 확인할 수 있다. `passwd` 명령의 권한을 보면 `s` 비트가 있어서 실행 시 effective UID가 root로 바뀐다.

```bash
ls -l /usr/bin/passwd               # -rwsr-xr-x ← s가 setuid
ls -l /usr/bin/wall                 # setgid tty 예시
cat /proc/$$/status | grep -E 'Uid|Gid|Groups'   # 현재 셸의 신원
```

`/proc/$$/status`의 출력을 보면 Uid/Gid 라인이 4개 숫자(Real, Effective, Saved, FS)로 표시되는 것을 확인할 수 있다. 위에서 설명한 4종 UID 모델의 직접적 증거다.

## 흔한 함정

신원 모델은 직관적으로 보이지만, 실제 운영에서 만나는 함정은 대부분 "당연하게 보이는 동작이 실제로는 다르게 작동하는" 종류다. 가장 흔한 첫 사고는 `usermod -G` 명령에서 일어난다. 이 명령은 supplementary group을 통째로 교체하는 동작이라서, 추가하려고 `usermod -G newgroup user`를 실행하면 기존에 가입한 모든 그룹이 한 번에 사라진다. 자동화 스크립트에서 이 실수가 일어나면 사용자가 갑자기 모든 협업 폴더에서 추방되는 사고로 이어진다. 추가하려면 반드시 `-aG` (append) 플래그를 명시해야 하며, 더 안전하게는 `gpasswd -a USER GROUP`처럼 한 번에 한 그룹씩 처리하는 명령을 쓰는 게 좋다.

비슷하게 명령 자체는 옳게 썼는데도 사고가 나는 경우가 있는데, 반영 시점의 비동기성 때문이다. `usermod -aG newgroup user`를 실행한 직후 `user`로 SSH가 연결되어 있어도 그 세션은 옛 그룹 정보를 그대로 유지한다. 커널이 프로세스의 GID set을 `setgroups()` 호출 시점에만 업데이트하기 때문이다. 새 SSH 연결을 맺거나 `newgrp` 명령으로 새 셸을 띄워야 변경이 반영되는데, 자동화 스크립트가 그룹 추가 직후 그 그룹 권한이 필요한 작업을 하려고 하면 권한 거부로 실패한다. 이 함정은 CI/CD 파이프라인에서 특히 자주 만난다.

규모가 큰 환경에서는 NGROUPS_MAX 한계가 또 다른 종류의 문제를 만든다. 이론상 한 사용자가 65536개의 supplementary group에 속할 수 있지만, NFS 같은 일부 프로토콜은 전통적인 16개 제한을 그대로 유지한다. 그룹이 많은 사용자가 NFS 마운트의 파일에 접근할 때 권한 거부가 일어나는데, 일반 도구로는 디버깅하기 매우 어려운 종류의 문제다 — 같은 시스템에서 로컬 파일 접근은 잘 되니까 권한 모델 자체를 의심하기 어렵다.

권한 모델 자체의 보안 가정을 깨는 함정도 있다. 대표적인 것이 setuid 셸 스크립트로, 거의 모든 OS가 이를 무시한다. setuid 비트를 셸 스크립트에 설정해도 실행 시점에 다른 스크립트로 바꿔치기되는 race condition이 있어서 커널이 setuid를 적용하지 않기 때문이다. setuid가 정말 필요하면 컴파일된 wrapper 바이너리를 만들고 그것이 스크립트를 호출하게 하거나, 더 일반적으로는 sudo 정책으로 대체하는 것이 표준이다.

`sudo`를 root와 동일하게 생각하는 것도 미묘한 오해다. sudo는 sudoers 정책 기반으로 effective UID를 변경하는 메커니즘이고, 모든 호출이 `/var/log/auth.log` (Debian) 또는 `/var/log/secure` (RHEL)에 기록된다. 권한 분리뿐 아니라 추적성 — 누가 언제 무엇을 했는지 — 을 제공한다는 점이 root 직접 로그인보다 sudo가 권장되는 핵심 이유다.

마지막으로 컨테이너 시대의 함정으로, 컨테이너 안의 root가 호스트에서 어떤 의미인지가 user namespace 설정에 따라 완전히 달라진다. user namespace를 사용하면 컨테이너 내부의 root(uid 0)가 호스트에서는 일반 사용자(예: uid 100000)로 매핑되어 격리가 보장된다. 하지만 이를 모르고 컨테이너 안에서 root라고 무엇이든 가능하다고 가정하면 보안 모델이 깨진다. 반대로 user namespace 없이 컨테이너를 실행하면 컨테이너 안의 root가 호스트에서도 진짜 root여서 컨테이너 escape 시 호스트 전체가 위협받는다.

## B1-1 매핑

이번 과제의 사용자·그룹 설계는 위 개념들의 직접적인 응용이다.

```
주민          primary gid     supplementary
agent-admin   agent-admin     agent-common, agent-core
agent-dev     agent-dev       agent-common, agent-core
agent-test    agent-test      agent-common
```

이 설계의 핵심 의도는 두 가지다. 우선 세 사용자 모두 `agent-common`에 속함으로써 공유 폴더(`upload_files`)에 자유롭게 접근할 수 있게 한다 — 이는 협업 영역을 만드는 표준 패턴이다. 동시에 `agent-test`만 `agent-core`에서 빠지도록 함으로써 핵심 데이터(`api_keys`)와 모니터링 로그(`/var/log/agent-app`)에 대한 접근을 차단한다. 테스트 사용자가 production 자격 증명에 접근하면 안 된다는 일반 보안 원칙의 단순한 적용이다.

monitor.sh 자체의 권한 설계도 이 모델을 활용한다. 파일 owner를 `agent-dev`로 두면 개발자가 수정 가능하고, group을 `agent-core`로 두면 admin이 cron으로 실행 가능하다 — admin도 core 멤버이므로 group 권한으로 실행권이 부여된다. 이 권한 설계가 다음 노트에서 다룰 file permissions와 ACL의 실용 사례가 된다.

## 인접 토픽

신원 모델의 응용을 더 깊이 이해하려면 권한 분리·격리·인증의 세 축으로 정리해 보면 좋다.

권한 분리 측면의 핵심은 POSIX capabilities다. 전통적으로 root는 "전부 가능"의 이분법이었지만, capabilities는 root의 능력을 39개 이상의 단위로 쪼갠다. 예를 들어 `CAP_NET_BIND_SERVICE`만 부여하면 그 프로세스는 1024 미만 포트에 바인딩만 할 수 있고 다른 root 능력은 갖지 못한다. setuid의 안전한 대안으로 권장되며, ping이 root setuid 없이 raw 소켓을 사용할 수 있는 이유도 capabilities 덕분이다.

격리 측면에서는 user namespace가 컨테이너 보안의 기반이다. 컨테이너 안의 UID와 호스트의 UID를 매핑할 수 있어서, 컨테이너 안의 root가 호스트에서는 일반 사용자가 된다. `/proc/PID/uid_map`에서 매핑을 직접 확인할 수 있고, rootless container의 핵심 기술이기도 하다. Docker가 비교적 늦게 user namespace를 기본 지원한 데 비해, Podman은 처음부터 rootless 지향으로 설계되었다는 점이 흥미롭다.

인증 측면에서는 PAM (Pluggable Authentication Modules)이 인증 단계의 모듈식 구성을 제공한다. SSH 로그인, sudo, screen unlock 등이 모두 PAM을 거치며, `/etc/pam.d/`의 설정으로 정책을 바꿀 수 있다. 2FA, 패스워드 강도 검사, 로그인 횟수 제한 같은 모든 인증 강화가 PAM 모듈로 구현되므로, 보안 강화 작업은 거의 항상 PAM 설정에서 시작된다.

새로운 모델로 systemd-homed가 흥미로운데, 사용자 홈을 LUKS 암호화된 portable 형태로 관리해서 사용자 정보가 `/etc/passwd`가 아닌 홈 디렉토리 자체에 보존된다. 노트북 도난 시 데이터 보호나 외장 디스크로 사용자 이동 같은 시나리오를 자연스럽게 지원한다. 마지막으로 ACL은 owner/group/other 모델로 표현 불가능한 권한 요구를 다루는 확장이다 — "한 디렉토리에 group A는 RW, group B는 R" 같은 요구가 ACL 없이는 표현 불가능한데, 다음 노트에서 자세히 다룬다.

## 참고

- `man 5 passwd`, `man 5 shadow`, `man 5 group` — 파일 형식
- `man useradd`, `man usermod`, `man getent` — 명령 매뉴얼
- `man 7 credentials` — UID/GID 모델 전반
- `man 7 capabilities` — 권한 분리 모델
- `man 5 nsswitch.conf` — NSS 설정

---
출처: B1-1 (Layer 1.2) · 학습일: 2026-05-07
