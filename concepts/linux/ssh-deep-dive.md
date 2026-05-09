# SSH의 동작 원리

## 개요

SSH(Secure Shell)는 신뢰할 수 없는 네트워크 위에서 두 시스템 사이에 암호화된 양방향 채널을 만들고, 그 위에서 셸 접속·파일 전송·포트 포워딩·리모트 명령 실행 등을 수행하는 프로토콜이다. 1995년 텔넷·rsh 같은 평문 프로토콜의 보안 문제를 해결하기 위해 만들어졌고, 현재는 거의 모든 서버 운영의 기본 입출구다. RFC 4251~4256으로 표준화되어 있으며, OpenSSH가 사실상의 reference 구현이다.

이번 과제에서 SSH 포트를 22 → 20022로 변경하고 root 원격 접속을 차단하는 것은 단순한 설정 변경처럼 보이지만, 그 이유와 실제 동작을 정확히 이해해야 명세의 *왜*에 답할 수 있다.

## 왜 알아야 하나

서버 운영의 첫 단계가 SSH 보안인 이유는, 인터넷에 노출된 SSH 데몬이 끊임없는 brute-force 공격의 대상이기 때문이다. AWS·GCP에 EC2 인스턴스 하나를 띄우고 22 포트를 열어두면 분 단위로 random IP에서 root·admin·user 같은 일반 사용자명으로 비밀번호 시도가 들어온다. `/var/log/auth.log` 또는 `/var/log/secure`에서 이 공격 흔적을 직접 볼 수 있다.

SSH의 정확한 동작 모델을 이해해야 하는 두 번째 이유는 디버깅이다. "키가 안 먹는다", "Permission denied (publickey)", "Connection closed by remote host" 같은 흔한 에러는 SSH 핸드셰이크의 어느 단계에서 실패하는지를 모르면 추측 디버깅으로 빠진다. 또한 키 기반 인증, agent forwarding, port forwarding 같은 고급 기능은 모두 같은 프로토콜 위에 쌓인 응용이라, 기본기가 없으면 실수로 보안 구멍을 만든다.

## SSH의 3단계 핸드셰이크

SSH 연결은 크게 세 단계로 구성된다. `ssh -vv user@host`로 직접 추적해 보면 이 흐름이 명확히 보인다.

```
1. Transport Layer       — 키 교환 + 채널 암호화 수립
   │
   ├─ 알고리즘 협상 (cipher, MAC, KEX)
   ├─ Diffie-Hellman 키 교환
   ├─ 서버 호스트 키 검증 (~/.ssh/known_hosts)
   └─ session key 생성 → 이 시점부터 모든 통신 암호화
   │
   ▼
2. User Authentication   — 클라이언트가 자기를 증명
   │
   ├─ publickey   (~/.ssh/id_*.pub vs 서버의 ~/.ssh/authorized_keys)
   ├─ password    (사용자 입력 vs /etc/shadow)
   ├─ keyboard-interactive (PAM 모듈 — 2FA 등)
   └─ gssapi/hostbased (드뭄)
   │
   ▼
3. Connection             — 채널 다중화 후 실제 작업
   │
   ├─ session   (셸·exec)
   ├─ direct-tcpip   (-L 포워딩)
   ├─ forwarded-tcpip (-R 역포워딩)
   └─ x11        (X 디스플레이 포워딩)
```

이 3단계 구조의 핵심 통찰은 **암호화 채널이 인증보다 먼저 수립된다**는 점이다. 클라이언트가 비밀번호를 보낼 때는 이미 채널이 암호화되어 있어 평문이 네트워크에 노출되지 않는다. 텔넷·FTP가 평문으로 인증 정보를 보내던 시대를 생각하면 큰 발전이다.

## 키 기반 인증의 실제 동작

SSH의 가장 큰 가치 중 하나가 키 기반 인증이다. 비밀번호 없이도 안전하게 접속할 수 있고, 자동화 스크립트(CI/CD, Ansible 등)의 기반이 된다. 동작 흐름은 다음과 같다.

```
[클라이언트]                         [서버]
  ~/.ssh/id_ed25519 (개인키)           ~/.ssh/authorized_keys
                                        (등록된 공개키 목록)

  1. "이 사용자명으로 로그인할게,
     이 공개키 등록되어 있어?"           ──→ authorized_keys 검색
                                        ←── "그 공개키 있음, challenge 줄게"
                                            (랜덤 데이터 + 세션 ID)
  2. 개인키로 challenge에 서명           ──→ 서명 검증 (공개키로)
                                        ←── 서명 OK → 인증 성공
```

핵심 보안 속성은 **개인키가 네트워크에 절대 전송되지 않는다**는 점이다. 서버는 공개키만 가지고 있고, 클라이언트는 개인키로 challenge에 서명해서 보낸다. 서명 검증은 공개키로 가능하지만 서명 생성은 개인키로만 가능하므로, 공격자가 채널을 도청해도 키를 훔칠 수 없다.

키 알고리즘은 시대에 따라 달라졌다. 과거에는 RSA가 표준이었지만, 현재는 Ed25519가 권장된다 — 더 작은 키 크기(256비트)로 RSA 4096비트 이상의 보안 강도를 제공하며 서명 생성·검증도 빠르다. `ssh-keygen -t ed25519`로 생성하는 게 현대적 default다.

## 호스트 키와 known_hosts

서버 인증은 또 다른 방향에서 이루어진다. 클라이언트가 서버를 신뢰한다는 보장이 필요한데, 이는 서버의 호스트 키와 클라이언트의 `~/.ssh/known_hosts`로 처리된다.

서버는 자기 호스트 키 쌍을 `/etc/ssh/ssh_host_*_key`(개인)와 `/etc/ssh/ssh_host_*_key.pub`(공개)에 가지고 있다. 클라이언트가 처음 접속할 때 서버는 자기 공개 호스트 키를 보내고, 클라이언트는 이걸 `~/.ssh/known_hosts`에 저장한다. 두 번째 접속부터는 받은 호스트 키가 known_hosts와 일치하는지 검증해서, 일치하지 않으면 다음과 같은 경고가 나온다.

```
WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!
Someone could be eavesdropping on you right now (man-in-the-middle attack)!
```

이 경고는 두 가지 상황에서 발생한다 — (1) 진짜 MITM 공격, (2) 서버 재설치 등으로 호스트 키가 정당하게 바뀐 경우. 후자가 훨씬 흔하지만 전자의 가능성이 있어 SSH는 항상 경고한다. 정당한 변경이라면 `ssh-keygen -R hostname`으로 known_hosts에서 해당 항목을 제거 후 재접속한다.

## 한 번 보자

SSH의 동작을 직접 관찰하려면 `-v` 플래그가 강력하다. 핸드셰이크 단계마다 어떤 정보를 주고받는지 보여주므로, 거의 모든 SSH 디버깅의 첫 번째 도구다.

```bash
ssh -v user@host                     # 기본 verbose
ssh -vv user@host                    # 더 자세 (알고리즘 협상까지)
ssh -vvv user@host                   # 최대 verbose (대부분 디버깅에 필요한 것 다)
```

키 생성과 등록의 표준 흐름은 다음과 같다.

```bash
# 클라이언트에서: 키 쌍 생성
ssh-keygen -t ed25519 -C "your_email@example.com"
# 기본 위치: ~/.ssh/id_ed25519 (개인키), ~/.ssh/id_ed25519.pub (공개키)

# 서버에 공개키 등록 (편한 방법)
ssh-copy-id user@host
# 내부적으로: 공개키를 서버의 ~/.ssh/authorized_keys에 추가

# 또는 수동으로
cat ~/.ssh/id_ed25519.pub | ssh user@host \
    'mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys'

# 키로 접속 테스트
ssh -i ~/.ssh/id_ed25519 user@host
```

권한 검증도 자주 필요하다. SSH는 보안상 매우 엄격해서, `~/.ssh/`나 `authorized_keys`의 권한이 너무 느슨하면 키 인증을 거부한다.

```bash
ls -la ~/.ssh/
# 기대: drwx------ (~/.ssh 자체)
#       -rw------- (id_ed25519, authorized_keys)
#       -rw-r--r-- (id_ed25519.pub)

chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519 ~/.ssh/authorized_keys
chmod 644 ~/.ssh/id_ed25519.pub
```

## 흔한 함정

SSH 운영에서 자주 만나는 함정은 보안 가정과 권한 관리에 집중되어 있다. 가장 자주 부딪히는 것이 키 권한 함정인데, SSH는 `~/.ssh/`나 `authorized_keys`의 권한이 group/other에게 열려 있으면 보안 위험으로 판단해 키 인증을 거부한다. 이때 에러 메시지가 평범해서("Permission denied (publickey)") 권한이 원인인지 알아채기 어려운데, `ssh -v`로 verbose 모드를 켜면 "Authentications that can continue"이나 "key permissions are too open" 같은 단서가 나온다. 비슷한 권한 함정으로 home 디렉토리 자체의 권한이 너무 열려 있어도 SSH가 키 인증을 거부할 수 있는데, group/other가 home에 쓸 수 있으면 누가 .ssh를 바꿔치기할 위험이 있기 때문이다 — `chmod o-w $HOME` 같은 강화가 필요하다.

서버 측 함정으로는 sshd_config 변경 후 데몬 재시작을 잊는 경우가 흔하다. 설정 파일을 수정해도 sshd가 메모리에 로드한 옛 설정으로 동작하므로 `systemctl reload sshd` 또는 `systemctl restart sshd`가 필요한데, 이를 잊으면 "내가 분명히 변경했는데 왜 안 먹지" 하며 시간을 낭비하게 된다. 더 위험한 것은 잘못된 설정으로 SSH를 막아버리는 시나리오로, 원격 서버에서 SSH 설정을 바꾸다가 재시작 후 접속이 안 되면 콘솔로 가야 하는 상황이 된다. 안전한 패턴은 (1) 새 SSH 세션을 다른 터미널에서 미리 열어두고 (2) 설정 변경 후 그 새 세션이 살아있는지 확인하고 (3) 첫 세션을 끊는 것으로, 이 절차로 lockout 사고를 거의 모두 막을 수 있다.

키 기반 인증을 설정해 놓고도 여전히 비밀번호를 묻는 경우도 자주 만난다. 보통 `authorized_keys`의 위치 또는 권한이 잘못되었거나, sshd_config의 `AuthorizedKeysFile`이 다른 경로를 가리키거나, 사용자명이 다르기 때문이다. `ssh -v`의 출력에서 "Trying private key: ..." 라인 다음에 "Server accepts key" 또는 "Permission denied"가 나오는데, 후자라면 서버 측 문제이며 sshd 로그(`journalctl -u sshd`)에 "Authentication refused: bad ownership or modes" 같은 명확한 메시지가 남아 있다.

마지막으로 호스트 키 변경 경고를 무시하는 습관은 위험하다. 진짜 MITM 공격일 가능성이 있으므로, 변경 이유가 명확하지 않으면(서버 재설치 알림 등) 의심해야 한다. 정당한 변경이라면 `ssh-keygen -R hostname`으로 known_hosts에서 제거 후 재접속하면서 새 fingerprint가 예상한 것인지 확인하는 절차가 필요하다.

## B1-1 매핑

이번 과제에서 SSH 관련 요구는 두 가지다 — 포트 22 → 20022 변경, root 원격 접속 차단. 두 변경 모두 `/etc/ssh/sshd_config`에서 처리하는데, 자세한 옵션 의미는 다음 노트(`sshd-config.md`)에서 다룬다.

여기서 짚어둘 점은 이 두 변경의 *왜*다. 포트 변경의 보안 가치는 종종 "security through obscurity"라며 평가절하되지만, 실제 효과는 명확하다. 22 포트는 자동 brute-force 봇이 끊임없이 두드리는 표적이고, 임의의 high-port로 옮기면 random scanning이 그 포트를 발견할 확률이 매우 낮아져 로그 노이즈가 극적으로 줄어든다. 결정된 공격자(targeted attack)에게는 nmap 한 번이면 들통나지만, 자동화된 background noise는 거의 사라진다. 이 둘을 구분해서 이해하는 게 정확하다.

root 원격 접속 차단(`PermitRootLogin no`)의 가치는 더 명확하다. root 계정은 모든 시스템에 존재하는 알려진 사용자명이라 brute-force의 첫 번째 대상이 되고, 만약 비밀번호 인증이 활성화되어 있다면 단 한 번의 추측으로 시스템 전체가 위협받을 수 있다. 일반 사용자로 로그인 후 sudo로 권한을 얻는 패턴이 (1) 사용자명 자체가 비공개 정보라서 표적 면적이 줄고 (2) sudo 사용이 모두 감사 로그에 남으며 (3) sudoers 정책으로 세밀한 권한 제어가 가능하다는 세 가지 이점을 가진다.

monitor.sh는 SSH 자체와 직접 관련은 없지만, 이번 과제에서 SSH 데몬이 정상 동작하는지가 verify.sh의 검증 항목이라 `ss -tulnp | grep 20022`로 LISTEN 상태를 확인하게 된다.

## 인접 토픽

SSH의 응용은 자동화·터널링·고급 인증의 세 축으로 정리해 볼 수 있다.

자동화 측면의 핵심은 ssh-agent다. 매번 키 비밀번호(passphrase)를 입력하지 않도록 agent가 메모리에 키를 보관하고, agent forwarding(`ssh -A`)으로 원격 서버에서도 로컬 키를 사용할 수 있게 해준다. CI/CD 파이프라인이나 다단계 jump host 환경에서 핵심 도구지만, agent forwarding은 보안 위험도 있다 — 중간 서버가 손상되면 그 서버의 root가 클라이언트의 모든 SSH 세션을 hijack할 수 있다. ProxyJump(`-J host1`)는 더 안전한 대안으로, 중간 서버에 키를 노출하지 않고 직접 최종 호스트에 연결한다.

터널링 측면에서는 SSH가 임의의 TCP 트래픽을 암호화 채널로 전달할 수 있다는 점이 강력하다. local forwarding(`-L`)은 로컬 포트를 원격 서비스에 연결, remote forwarding(`-R`)은 그 반대 방향, dynamic forwarding(`-D`)은 SOCKS 프록시를 만든다. 방화벽을 우회하거나 안전하지 않은 네트워크에서 회사 내부 서비스에 접속할 때 자주 쓰는 패턴이다.

고급 인증 측면에서는 SSH 인증서(certificate)가 흥미롭다. 일반 SSH 키는 클라이언트가 서버의 `authorized_keys`에 일일이 등록되어야 하지만, SSH CA(Certificate Authority)를 두면 단일 CA 키로 서명한 인증서가 모든 서버에서 인정된다. 대규모 인프라에서 키 관리의 게임체인저로 평가받으며, HashiCorp Vault, Smallstep CA, Teleport 같은 도구가 이 영역을 다룬다.

마지막으로 mosh(mobile shell)는 SSH의 한계를 보완한 후속 프로토콜이다. SSH는 TCP 위에 있어 네트워크가 끊기면 세션이 죽지만, mosh는 UDP 기반에 SSP(State Synchronization Protocol)를 사용해 IP가 바뀌어도(Wi-Fi → 4G 등) 세션이 유지된다. 모바일·여행 환경에서 매우 유용하다.

## 참고

- `man ssh`, `man sshd`, `man ssh-keygen`, `man ssh-agent`
- RFC 4251~4256 — SSH 프로토콜 표준
- `ssh -v` / `-vv` / `-vvv` — 핸드셰이크 디버깅의 첫 번째 도구
- [OpenSSH Cookbook](https://en.wikibooks.org/wiki/OpenSSH/Cookbook) — 다양한 운영 패턴

---
출처: B1-1 (Layer 2.1) · 학습일: 2026-05-09
