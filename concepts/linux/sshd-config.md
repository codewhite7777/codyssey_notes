# sshd_config — SSH 데몬 설정

## 개요

`sshd_config`는 OpenSSH 서버 데몬(`sshd`)의 동작을 제어하는 핵심 설정 파일로, `/etc/ssh/sshd_config`에 위치한다. 한 줄에 한 옵션의 KEY VALUE 형식으로 구성되며, 주석은 `#`으로 시작한다. 서버의 보안 정책 — 어떤 포트로 받을지, 누가 접속 가능한지, 어떤 인증 방식을 허용할지, 어떤 기능을 켤지 — 가 모두 이 한 파일로 결정된다.

이번 과제의 SSH 관련 요구사항(포트 변경, root 차단)은 모두 `sshd_config`의 한두 줄 변경으로 처리된다. 단순해 보이지만, 각 옵션의 의미와 상호작용을 정확히 이해해야 의도하지 않은 보안 구멍이나 잠금(lockout)을 피할 수 있다.

## 왜 알아야 하나

`sshd_config`의 옵션은 100개가 넘고 대부분 보안에 직결된다. 잘못 설정하면 (1) 보안이 약해지거나, (2) 본인이 접속 못 하게 되거나, (3) 의도하지 않은 동작을 한다. 가장 흔한 사고는 옵션 변경 후 데몬을 재시작하지 않거나, 잘못된 설정으로 재시작 후 접속이 안 되는 것이다.

또 한 가지 중요한 이유는 보안 감사·표준 준수다. CIS Benchmark, PCI-DSS, HIPAA 같은 보안 표준은 모두 sshd_config에 대한 구체적 권고사항을 가지고 있다 — 예를 들어 "PermitRootLogin no", "Protocol 2", "PasswordAuthentication no" 같은 설정이 표준의 default다. 운영자라면 이런 권고사항의 *이유*를 설명할 수 있어야 한다.

## 설정 파일 구조와 우선순위

`sshd_config`는 평범한 텍스트 파일이지만, 우선순위 규칙이 한 가지 있다 — **첫 번째로 발견된 값이 적용된다**. 같은 옵션이 두 번 나오면 위의 것이 우선이고 아래는 무시된다. 이는 일반적인 "마지막 값이 이김" 직관과 반대라서 함정이 되기도 한다.

```
# /etc/ssh/sshd_config 의 일반적 구조

# 1. 네트워크 설정
Port 22
ListenAddress 0.0.0.0
AddressFamily any

# 2. 호스트 키
HostKey /etc/ssh/ssh_host_rsa_key
HostKey /etc/ssh/ssh_host_ed25519_key

# 3. 인증 설정
PermitRootLogin prohibit-password
MaxAuthTries 6
PubkeyAuthentication yes
PasswordAuthentication yes
ChallengeResponseAuthentication no
UsePAM yes

# 4. 사용자 제한
AllowUsers alice bob
AllowGroups sshusers

# 5. 기능 토글
X11Forwarding no
PrintMotd no
TCPKeepAlive yes
ClientAliveInterval 300

# 6. Subsystem
Subsystem sftp /usr/lib/openssh/sftp-server

# 7. Match 블록 (조건부 설정)
Match User git
    PasswordAuthentication no
    AllowTcpForwarding no
```

`Match` 블록은 특정 조건(사용자, 그룹, 호스트, 주소 등)에만 적용되는 설정을 만들 수 있다. 위 예에서 `Match User git`은 git 사용자에 한해 비밀번호 인증과 TCP 포워딩을 끈다. Git 호스팅 서비스가 흔히 사용하는 패턴이다.

## 핵심 옵션의 의미

이번 과제와 보안 운영에 가장 자주 만지는 옵션들의 정확한 동작을 짚어보자.

`Port`는 sshd가 LISTEN할 TCP 포트를 지정한다. 기본값은 22이며, 여러 줄로 여러 포트를 동시에 열 수도 있다. 1024 미만 포트는 root 권한이 필요하지만 sshd는 보통 root로 시작하므로 문제없다.

`ListenAddress`는 어떤 네트워크 인터페이스에서 받을지 지정한다. 기본 `0.0.0.0`은 모든 IPv4 인터페이스, `::`은 모든 IPv6, `127.0.0.1`은 로컬호스트만, `192.168.1.10`은 특정 인터페이스만 등이다. 내부 네트워크에서만 SSH를 받고 싶으면 이걸로 외부 인터페이스를 차단할 수 있다.

`PermitRootLogin`은 가장 자주 만지는 보안 옵션이다. 가능한 값들의 의미가 미묘하게 다른데, `yes`는 모든 root 접속 허용(거의 비추), `no`는 root 접속 완전 차단, `prohibit-password`는 키 기반은 OK이지만 비밀번호는 X(modern Ubuntu의 default), `forced-commands-only`는 root는 forced command만 허용한다(자동화 백업 등). 이번 과제에서 요구하는 "Root 원격 차단"은 `no`로 설정한다 — `prohibit-password`도 보안적으로는 강하지만(키 인증만 허용) 명세 의도가 더 명확한 차단이라면 `no`가 정답이다.

`PasswordAuthentication`은 비밀번호 인증을 허용할지 결정한다. brute-force 공격에 대한 방어 측면에서 `no`로 설정하고 키 인증만 강제하는 것이 보안 강한 환경의 표준이지만, 키를 잃으면 복구가 까다로우므로 신중해야 한다. `PubkeyAuthentication`은 공개키 인증의 활성화로, 거의 항상 `yes`다.

`AuthorizedKeysFile`은 공개키 등록 위치다. 기본 `.ssh/authorized_keys`인데, 절대 경로 또는 home-relative 경로로 변경 가능하며 `%u`는 사용자명, `%h`는 home 경로로 치환된다. 중앙 집중식 키 관리에 활용한다.

`AllowUsers` / `AllowGroups` / `DenyUsers` / `DenyGroups`는 SSH 접속 가능한 사용자·그룹을 화이트/블랙리스트로 제한한다. 검사 순서는 DenyUsers → AllowUsers → DenyGroups → AllowGroups로, AllowUsers를 한 번이라도 명시하면 거기 없는 사용자는 모두 차단된다는 점이 함정이 되기도 한다.

`MaxAuthTries`는 한 연결에서 시도할 수 있는 인증 횟수다. 기본 6인데, brute-force 방어로 3 정도로 줄이는 것이 흔한 강화 패턴이다. `ClientAliveInterval`과 `ClientAliveCountMax`는 idle 세션 관리로, interval 초마다 keepalive를 보내고 count 번 응답이 없으면 끊는다 — 네트워크 끊김으로 인한 좀비 세션을 방지한다.

`X11Forwarding`, `AllowTcpForwarding`, `PermitTunnel`은 SSH의 부가 기능을 켜고 끈다. 보안 강한 환경에서는 모두 `no`로 두고 필요할 때만 활성화하는 게 표준이다.

## 변경 후 적용 방법

`sshd_config`를 수정해도 즉시 적용되지 않는다. sshd 데몬이 메모리에 로드한 옛 설정으로 동작하므로 명시적 재로드가 필요하다.

```bash
sudo systemctl reload sshd     # 기존 연결 유지하며 설정 재로드 (선호)
sudo systemctl restart sshd    # 데몬 완전 재시작 (기존 연결 끊김 가능)
```

`reload`는 `restart`보다 안전한데, 기존 SSH 세션을 끊지 않고 새 설정을 적용한다. 가능하면 항상 `reload`를 사용한다.

변경 전에 문법 검증도 가능하다. `sshd -t`는 설정 파일을 파싱해서 문법 오류를 확인하며(데몬을 시작하지는 않음), 잘못된 설정으로 sshd가 시작 안 되는 사고를 막는 안전장치다. `sshd -T`는 더 유용한데, Match 블록·기본값 등을 모두 반영한 효과적 설정 전체를 보여줘서 "내가 변경한 옵션이 정말 적용되는가"를 확인할 때 쓴다.

```bash
sudo sshd -t                   # 문법 검증 (출력 없으면 OK)
sudo sshd -T                   # 실제 적용될 효과적 설정 전체 출력
```

## 한 번 보자

이번 과제의 변경을 단계적으로 따라가 보자. 먼저 변경 전 상태 확인이다.

```bash
# 현재 sshd가 LISTEN하는 포트
sudo ss -tulnp | grep sshd

# 현재 적용된 효과적 설정 (가장 신뢰)
sudo sshd -T | grep -E '^(port|permitrootlogin|passwordauthentication)'

# 설정 파일 직접 보기
sudo grep -E '^#?(Port|PermitRootLogin|PasswordAuthentication)' /etc/ssh/sshd_config
```

설정 변경은 `sed`로 멱등하게 처리할 수 있다. `^#?`로 주석 처리된 줄도 함께 매칭하면, 새로 추가하는 게 아니라 항상 같은 줄을 정확히 한 번 갱신한다 — 멱등성이 핵심이다.

```bash
# Port를 20022로 변경 (주석 처리된 줄도 처리)
sudo sed -i 's/^#\?Port .*/Port 20022/' /etc/ssh/sshd_config

# Root 원격 접속 차단
sudo sed -i 's/^#\?PermitRootLogin .*/PermitRootLogin no/' /etc/ssh/sshd_config

# 변경 검증
sudo grep -E '^(Port|PermitRootLogin)' /etc/ssh/sshd_config
```

문법 검증 후 reload한다. 안전한 운영 패턴은 다음과 같다.

```bash
# 1. 문법 검증 (오류면 즉시 종료, 적용 X)
sudo sshd -t || { echo "syntax error in sshd_config"; exit 1; }

# 2. ★ 다른 터미널에서 새 SSH 세션 미리 열어두기 (실수 방어)

# 3. 데몬 재로드
sudo systemctl reload sshd

# 4. 변경 검증 — 새 포트로 접속 가능한지
ssh -p 20022 user@host

# 5. 옛 포트는 막혔는지 확인
sudo ss -tulnp | grep -E ':22|:20022'   # 22는 없어야, 20022는 LISTEN
```

## 흔한 함정

sshd_config 운영에서 가장 자주 부딪히는 함정은 변경 후 재시작을 잊는 것이다. 설정 파일 수정만 하고 sshd가 옛 설정으로 동작 중인데 "내 변경이 안 먹는다"고 디버깅하는 시나리오가 흔하며, `sshd -T`로 효과적 설정을 확인하는 습관이 이를 방지한다. 비슷한 맥락에서 sed로 일부분만 변경하다 옛 줄이 남아 있는 경우도 자주 있다 — `Port 22` 주석 처리된 줄과 `Port 20022` 새 줄이 둘 다 파일에 있으면, 첫 번째로 발견된 값이 적용되므로 어느 쪽이 위에 있는지에 따라 결과가 달라진다. sed로 변경할 때 주석 포함 모든 매칭을 한 번에 처리하는 패턴(`s/^#\?Port .*/Port 20022/`)이 안전한 이유다.

원격 작업의 가장 위험한 함정은 잘못된 설정으로 SSH를 막아버리는 것이다. 예를 들어 `AllowUsers alice` 같은 줄을 추가했는데 alice가 잘못된 사용자명이거나, `PermitRootLogin no`를 적용했는데 root로만 접속하던 환경이거나 — 재시작 후 접속이 안 되면 콘솔로 가야 한다. 클라우드 인스턴스라면 프로비저닝 콘솔로, 물리 서버라면 데이터센터로 가야 한다는 의미다. 안전한 패턴은 (1) 변경 전에 다른 터미널에서 새 SSH 세션을 미리 열어두고, (2) 변경 후 그 세션이 살아있는지 확인하고, (3) 이상 없으면 첫 세션을 닫는 것으로, 이 절차로 잘못된 설정으로 인한 lockout을 거의 모두 막을 수 있다.

`Match` 블록의 들여쓰기와 범위도 함정이 된다. Match는 다음 Match가 나올 때까지 또는 파일 끝까지 적용되는데, 이를 모르고 글로벌 옵션을 Match 아래에 추가하면 그 Match에만 적용된다. 들여쓰기는 가독성용이고 실제로는 의미 없지만, 가독성을 잃으면 어디까지가 Match 범위인지 헷갈리기 쉽다.

마지막으로 `AuthorizedKeysFile`의 권한 함정이 키 인증 실패의 흔한 원인이다. SSH는 보안상 `~/.ssh/`나 `authorized_keys`의 권한이 group/other에게 열려 있으면 키 인증을 거부하는데, 이때 sshd 로그(`/var/log/auth.log` 또는 `journalctl -u sshd`)에 "Authentication refused: bad ownership or modes"라고 명시적으로 나온다. 항상 sshd 로그를 함께 봐야 정확한 원인을 알 수 있다.

## B1-1 매핑

이번 과제의 SSH 설정은 두 줄 변경으로 끝난다.

```
# /etc/ssh/sshd_config 변경 사항
Port 20022
PermitRootLogin no
```

명세에서 검증 방법으로 안내한 흐름은 다음과 같다.

```bash
# 1. 설정 파일에서 변경 확인
grep -E '^(Port|PermitRootLogin)' /etc/ssh/sshd_config

# 2. 데몬에 적용된 효과적 설정 확인
sudo sshd -T | grep -E '^(port|permitrootlogin)'

# 3. LISTEN 포트 확인
sudo ss -tulnp | grep sshd
# 기대: ... :::20022 ... users:(("sshd",pid=...))

# 4. 실제 새 포트로 접속 가능한지
ssh -p 20022 agent-admin@<host>
```

setup 스크립트(`setup/01-ssh.sh`)에 이 변경을 멱등하게 작성하고, verify 스크립트는 위 4단계를 자동 검증한다.

한 가지 운영 팁: OrbStack Linux Machine에서 작업할 때, 호스트(Mac)에서 `ssh codyssey-b1-1@orb`처럼 OrbStack 통합 SSH로 들어가면 포트 변경 효과가 잘 안 보일 수 있다. 명세의 의도는 "표준 SSH 데몬에서 포트 변경"이므로, 변경 후 `ssh -p 20022 agent-admin@<orb-internal-ip>` 형태로 직접 접속해 검증하는 게 정확하다.

## 인접 토픽

sshd_config의 응용은 보안 강화·접근 제어·통합 인증의 세 갈래로 정리해 볼 수 있다.

보안 강화 측면에서는 fail2ban이 자주 짝지어 사용된다. sshd 로그를 모니터링해서 일정 횟수 인증 실패한 IP를 자동으로 iptables/firewalld 룰로 차단하는 도구로, brute-force 공격을 효과적으로 줄인다. 단 IPv6 지원, 대규모 환경에서의 성능 등의 한계도 알아둘 가치가 있다.

접근 제어 측면에서는 jump host(bastion host) 패턴이 표준이다. 인터넷에 노출된 단 하나의 견고한 서버를 jump host로 두고, 모든 다른 서버는 jump host에서만 접속 가능하도록 방화벽으로 제한하는 방식이다. SSH의 `ProxyJump`(`-J host1`) 옵션이 이 패턴을 직접 지원하며, AWS Session Manager, GCP IAP 같은 클라우드 서비스가 jump host의 현대적 대체재로 쓰인다.

통합 인증 측면에서는 SSH 인증서(certificate)와 OpenID Connect 통합이 흥미롭다. 일반 SSH 키 관리는 사용자가 늘어나면 관리 부담이 커지는데, SSH CA로 발급한 단기 인증서를 사용하면 사용자 추가·제거가 인증서 발급 정책 한 곳에서 처리된다. HashiCorp Vault, Smallstep CA, Teleport 같은 도구가 이 영역을 다룬다.

마지막으로 sshd의 부속 기능들도 알아두면 좋다. SFTP는 파일 전송용 subsystem으로 `Subsystem sftp /usr/lib/openssh/sftp-server`로 활성화되며, ChrootDirectory와 결합해 격리된 SFTP-only 사용자를 만들 수 있다. SCP는 SFTP보다 단순한 파일 복사 프로토콜인데, 최근 OpenSSH는 SCP를 SFTP 위에 구현하는 방향으로 움직이고 있어(`scp -O` vs 새 default) 호환성 이슈를 만들기도 한다.

## 참고

- `man sshd_config` — 모든 옵션의 정식 정의
- `man sshd` — 데몬 자체
- `sshd -t` / `sshd -T` — 검증과 효과적 설정 출력
- [Mozilla OpenSSH Guidelines](https://infosec.mozilla.org/guidelines/openssh) — 보안 강화 설정 권고
- [CIS Benchmarks for SSH](https://www.cisecurity.org/) — 표준 준수 권고

---
출처: B1-1 (Layer 2.2) · 학습일: 2026-05-09
