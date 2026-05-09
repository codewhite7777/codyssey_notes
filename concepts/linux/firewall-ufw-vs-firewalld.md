# 방화벽: ufw와 firewalld

## 개요

Linux의 방화벽은 커널의 netfilter 프레임워크 위에 쌓인 규칙 집합으로, 들어오고 나가는 네트워크 패킷을 검사·필터링·변환하는 메커니즘이다. 직접 사용하는 도구는 `iptables`(전통)와 `nftables`(현대)인데, 이들은 강력하지만 문법이 복잡하고 규칙이 누적되면 관리가 어렵다. 그래서 더 간편한 사용자 인터페이스로 `ufw`(Uncomplicated Firewall)와 `firewalld` 두 가지 wrapper가 등장했다.

이번 과제는 ufw 또는 firewalld 중 하나를 선택해 활성화하고, TCP 20022(SSH)와 15034(앱)만 허용하는 정책을 구성하라는 요구다. 두 도구가 같은 일을 하지만 모델·문법이 다르므로 각각의 특징을 이해해야 적절히 선택하고 운영할 수 있다.

## 왜 알아야 하나

서버에 노출된 모든 포트는 잠재적 공격 표면이다. 방화벽 정책의 기본 원칙은 "기본 거부 + 명시적 허용"으로, 명세가 요구하는 "20022·15034만 허용"이 정확히 이 패턴이다. 이 원칙을 위반하면(예: `ufw default allow incoming`) 모든 포트가 인터넷에 노출되어 공격 표면이 폭증한다.

또한 방화벽은 다른 layer(앱 인증, OS 권한)와 함께 defense-in-depth를 구성한다. 앱이 실수로 안전하지 않게 작성되었더라도 방화벽이 그 포트를 막고 있으면 공격자가 도달조차 못 하므로, 운영의 최외곽 방어선 역할을 한다. SSH 자체가 매우 견고해도 22 포트가 인터넷에 열려 있으면 brute-force 시도가 끊임없이 들어오는데, 방화벽으로 신뢰된 IP만 허용하면 이 noise가 사라진다.

## netfilter·iptables·nftables — 그 위의 ufw·firewalld

방화벽 도구를 이해하려면 layer 구조를 먼저 짚어야 한다.

```
[사용자 도구]
   ufw      Ubuntu·Debian 표준, 단순 명령 인터페이스
   firewalld   RHEL·Fedora 표준, zone 기반 관리
        │
        ▼ (둘 다 아래 layer를 호출)
[규칙 관리]
   iptables  전통적 도구, 단일 표(table)에 chain·rule
   nftables  iptables의 후계자, 더 표현력 있는 문법
        │
        ▼
[커널]
   netfilter  실제 패킷 검사·필터링·변환을 수행하는 커널 프레임워크
```

ufw와 firewalld는 같은 일(netfilter 규칙 관리)을 다른 방식으로 추상화한다. 둘이 동시에 활성화되면 충돌이 생기므로 한 시스템에 하나만 사용하는 게 표준이다.

iptables는 사라진 게 아니다. ufw·firewalld가 내부적으로 iptables(또는 nftables) 명령을 호출해 규칙을 만들기 때문에 `iptables -L -v -n`으로 실제 적용된 규칙을 직접 볼 수 있다. 디버깅 시 자주 활용한다.

## ufw — 단순함의 미학

ufw는 Ubuntu가 2008년에 도입한 wrapper로, "복잡하지 않게(uncomplicated)"라는 이름에서 알 수 있듯 단순한 명령 인터페이스가 핵심 가치다. 기본적인 정책 — 들어오는 패킷 거부, 특정 포트만 허용 — 을 짧은 명령으로 표현한다.

```bash
sudo ufw default deny incoming      # 들어오는 패킷 기본 거부
sudo ufw default allow outgoing     # 나가는 패킷 기본 허용
sudo ufw allow 20022/tcp            # SSH 포트 허용
sudo ufw allow 15034/tcp            # 앱 포트 허용
sudo ufw enable                     # 방화벽 활성화 + 부팅 시 자동 시작

sudo ufw status verbose             # 현재 상태
sudo ufw status numbered            # 룰을 번호와 함께 출력 (삭제 시 유용)
sudo ufw delete <number>            # 특정 룰 삭제
```

ufw의 장점은 가독성이다. `ufw allow 22/tcp`는 누가 봐도 의도가 명확하다. 단점은 복잡한 규칙 — 예를 들어 "특정 인터페이스의 특정 IP 범위에서만 들어오는 ICMP 허용" 같은 것 — 을 표현하기 어렵다는 점인데, 그런 경우는 ufw도 결국 iptables 직접 편집(`/etc/ufw/before.rules`)으로 해결한다.

ufw 설정 파일은 `/etc/ufw/` 아래에 있다. `ufw.conf`가 메인이고, `before.rules`·`after.rules`로 ufw가 자동 생성하는 규칙 전후에 임의 iptables 규칙을 끼워넣을 수 있다.

## firewalld — zone 기반 모델

firewalld는 Red Hat이 2011년에 도입한 도구로, 모델이 더 정교하다. 핵심 개념이 zone인데, 네트워크 인터페이스를 신뢰 수준에 따라 그룹화하고 각 zone에 다른 정책을 적용한다.

기본 제공 zone들은 다음과 같다.

```
trusted    완전 신뢰 (모든 트래픽 허용)
home       가정 네트워크 (대부분 허용)
work       업무 네트워크 (적당히 허용)
public     공개 네트워크 (대부분 거부, default)
external   NAT 마스커레이드용
internal   내부 인터페이스용
dmz        DMZ — 외부 노출 호스트
block      모든 들어오는 패킷 거부
drop       block과 비슷하지만 응답조차 안 보냄
```

각 zone은 자기만의 허용 서비스·포트 목록을 가진다.

```bash
sudo firewall-cmd --get-default-zone        # 현재 default zone (보통 public)
sudo firewall-cmd --get-active-zones        # 인터페이스별 zone 매핑
sudo firewall-cmd --list-all                # default zone의 모든 정책

# 영구 규칙 추가
sudo firewall-cmd --permanent --add-port=20022/tcp
sudo firewall-cmd --permanent --add-port=15034/tcp
sudo firewall-cmd --reload                  # 설정 적용 (★)

# 임시 규칙 (재부팅 시 사라짐) — 테스트용
sudo firewall-cmd --add-port=20022/tcp
```

firewalld의 특징은 `--permanent`와 `--runtime`의 분리다. permanent는 디스크에 저장되어 재부팅 후에도 유지되고, runtime은 메모리에만 있어 즉시 적용되지만 재부팅 시 사라진다. 이 분리가 안전한 운영에 도움이 되는데, runtime으로 먼저 테스트한 후 permanent로 확정하는 패턴이 가능하다. 하지만 permanent로만 추가하고 reload를 잊으면 적용 안 된 채로 남는 함정이 있다.

서비스 이름으로도 허용 가능하다. firewalld는 well-known 서비스의 이름·포트를 미리 정의해 두고 있다.

```bash
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --permanent --add-service=http
```

## 어느 것을 선택할까

배포판이 결정해 주는 경우가 많다. Ubuntu/Debian은 ufw가 기본 설치되어 있고, RHEL/Fedora/Rocky는 firewalld가 기본이다. 명세가 "ufw 또는 firewalld 중 하나"라고 한 것은 본인 환경의 default를 그대로 쓰라는 의미로 읽힌다.

특성을 비교해 보면 ufw는 단일 호스트의 단순한 정책에 강하고, firewalld는 멀티 인터페이스·멀티 zone 환경(노트북이 회사·집·카페를 오가는 등)에 강하다. 이번 과제는 단일 서버에 두 포트만 허용하는 단순한 시나리오라 ufw가 더 자연스럽지만, firewalld도 동일하게 잘 동작한다.

OrbStack의 Ubuntu 22.04 머신은 ufw가 기본이라 ufw를 권장한다.

## 한 번 보자

ufw로 이번 과제 정책을 구성하는 흐름은 다음과 같다.

```bash
# 1. 현재 상태 확인 (방화벽 활성 여부)
sudo ufw status

# 2. 정책 설정 — 명시적 허용 외 모두 거부
sudo ufw default deny incoming
sudo ufw default allow outgoing

# 3. 필요한 포트만 허용
sudo ufw allow 20022/tcp comment 'SSH'
sudo ufw allow 15034/tcp comment 'agent-app'

# 4. 활성화 (★ 만약 SSH 세션 중이고 22로 들어와 있다면 끊김 주의)
sudo ufw enable

# 5. 상태 검증
sudo ufw status verbose
# 출력 예:
# Status: active
# Default: deny (incoming), allow (outgoing)
# To              Action      From
# 20022/tcp       ALLOW IN    Anywhere
# 15034/tcp       ALLOW IN    Anywhere
```

설정 후 실제 적용된 iptables 규칙도 확인할 수 있다 — ufw가 어떻게 iptables 규칙을 만드는지 보면 두 layer의 관계가 명확해진다.

```bash
sudo iptables -L -v -n | head -30
sudo iptables -L ufw-user-input -v -n   # ufw가 만든 user input chain
```

firewalld 환경이라면 다음과 같다.

```bash
# 활성화 + default zone 확인
sudo systemctl enable --now firewalld
sudo firewall-cmd --get-default-zone

# 포트 허용 (영구)
sudo firewall-cmd --permanent --add-port=20022/tcp
sudo firewall-cmd --permanent --add-port=15034/tcp
sudo firewall-cmd --reload

# 검증
sudo firewall-cmd --list-all
```

## 흔한 함정

방화벽 운영에서 가장 위험한 함정은 SSH 세션을 끊어버리는 것이다. SSH로 원격 접속한 상태에서 22 포트를 차단하거나 default deny를 적용하기 전에 SSH 포트를 허용하지 않으면, 활성화 직후 본인 세션이 즉시 끊긴다. 클라우드 인스턴스라면 콘솔로 가야 복구할 수 있고, 그 사이 본인이 자기 서버에 접근 못 하는 상황이 된다. 안전한 패턴은 (1) SSH 포트 허용을 먼저 추가하고 (2) 다른 터미널에서 새 SSH 세션 미리 열어두고 (3) 그 다음 활성화하고 (4) 새 세션이 살아있는지 확인하는 것이다.

비슷한 맥락에서 ufw와 firewalld가 동시 활성화되어 충돌하는 함정도 있다. 둘이 같은 netfilter 위에서 작동하므로 어느 한쪽이 실수로 활성화되면 규칙이 예상과 다르게 적용된다. 한 시스템에 하나만 두는 게 원칙이며, 다른 도구로 옮길 때는 기존 도구를 명시적으로 비활성화해야 한다.

규칙 적용 시점도 함정이 된다. firewalld의 `--permanent` 옵션은 디스크에만 저장하고 runtime에는 적용하지 않으므로, `--reload`를 잊으면 변경이 안 먹힌다. 반대로 `--permanent` 없이 추가한 규칙은 재부팅 후 사라진다. 운영 작업에서는 항상 `--permanent + --reload` 조합으로 영구 변경을 명시해야 한다.

규칙 순서도 미묘한 함정이다. iptables는 chain의 규칙을 위에서 아래로 평가하면서 첫 번째 매치에서 결정을 내린다. ufw는 자동으로 적절한 위치에 규칙을 삽입하지만, 직접 iptables를 편집하는 경우 거부 규칙 뒤에 허용 규칙을 두면 허용이 적용 안 된다. `iptables -L -v -n --line-numbers`로 실제 순서를 확인하는 습관이 필요하다.

컨테이너 환경에서는 또 다른 종류의 함정이 등장한다. Docker는 iptables 규칙을 자기 마음대로 추가·수정하므로 ufw·firewalld 정책을 우회할 수 있다. 호스트 방화벽이 80 포트를 막아도 `docker run -p 80:80`은 그 정책을 무시하고 외부 노출을 만든다. 이를 방지하려면 Docker의 `iptables=false` 설정 또는 `DOCKER-USER` chain에 명시적 규칙 추가가 필요하다.

마지막으로 클라우드 환경의 보안 그룹과 호스트 방화벽이 두 layer로 작동한다는 점을 기억해야 한다. AWS Security Group, GCP Firewall은 호스트 외부의 네트워크 layer 방화벽이고, ufw·firewalld는 호스트 내부 layer다. 둘 다 통과해야 트래픽이 흐르므로 어느 한쪽만 열어도 도달 안 된다. 디버깅 시 두 layer를 모두 확인해야 한다.

## B1-1 매핑

이번 과제의 방화벽 정책은 단순하다 — 인바운드 기본 거부 + 20022·15034만 허용. ufw로 구성하면 다음 setup 스크립트가 된다.

```bash
# setup/02-firewall.sh
#!/bin/bash
set -euo pipefail

# 멱등성 — 이미 활성화되어 있어도 안전
sudo ufw --force reset                           # 기존 규칙 초기화
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 20022/tcp comment 'SSH'
sudo ufw allow 15034/tcp comment 'agent-app'
sudo ufw --force enable                          # --force로 confirm 프롬프트 우회

echo "[OK] firewall configured"
sudo ufw status verbose
```

verify 스크립트는 다음 항목을 자동 검증한다.

```bash
# verify.sh 단편
sudo ufw status | grep -q "Status: active" || { echo "[FAIL] ufw inactive"; exit 1; }
sudo ufw status | grep -q "20022/tcp.*ALLOW" || { echo "[FAIL] 20022 not allowed"; exit 1; }
sudo ufw status | grep -q "15034/tcp.*ALLOW" || { echo "[FAIL] 15034 not allowed"; exit 1; }
echo "[OK] firewall policy correct"
```

OrbStack Linux Machine 환경에서 한 가지 주의점이 있다. OrbStack은 Mac 호스트와 통합 네트워킹을 제공해서 일부 트래픽이 호스트 방화벽 layer를 우회할 수 있다. 명세 의도는 "Linux 표준 방화벽 정책 구성"이므로, ufw 활성화와 정책 적용 자체가 검증 대상이며 OrbStack의 외부 통합은 별개로 취급한다.

## 인접 토픽

방화벽의 응용은 표현력·관측·격리의 세 갈래로 정리해 볼 수 있다.

표현력 측면의 후속이 nftables다. iptables의 한계 — 여러 프로토콜을 위한 별도 도구(iptables, ip6tables, ebtables, arptables), 복잡한 chain 구조 — 를 해결한 후속 프레임워크로, 단일 도구로 IPv4·IPv6·ARP·bridge 트래픽을 모두 다룬다. 문법도 더 표현력 있고 일관적이며, 현대 Linux 배포판은 점진적으로 nftables로 이동하고 있다. ufw·firewalld도 백엔드를 nftables로 바꾸는 중이다.

관측 측면에서는 eBPF 기반 방화벽이 새 패러다임이다. Cilium, Calico 같은 도구가 eBPF로 패킷 필터링을 수행하는데, 커널에 동적으로 코드를 주입하므로 iptables보다 빠르고 더 풍부한 정책 표현이 가능하다. Kubernetes 네트워킹의 사실상 표준이 되어가고 있다.

격리 측면에서는 network namespace와 결합된 방화벽이 컨테이너의 기반이다. 각 namespace가 자기만의 iptables/nftables 규칙을 가지므로, 같은 호스트의 다른 컨테이너끼리도 방화벽으로 격리할 수 있다. Docker의 default bridge 네트워크가 정확히 이 패턴이다.

마지막으로 클라우드 환경의 layered 방화벽도 알아둘 가치가 있다. VPC level의 NACL(Network ACL), 인스턴스 level의 보안 그룹, 호스트 내부의 ufw·firewalld가 세 layer로 작동하며, 각각 stateful/stateless·deny-only/allow-only 등 미묘한 차이가 있다. 운영자는 이 layer들의 상호작용을 이해해야 정확한 트래픽 디버깅이 가능하다.

## 참고

- `man ufw` — ufw 사용법
- `man firewalld`, `man firewall-cmd` — firewalld 사용법
- `man iptables`, `man nftables` — 하부 도구
- [Netfilter Project](https://www.netfilter.org/) — 커널 프레임워크 공식 사이트
- [Ubuntu UFW Guide](https://help.ubuntu.com/community/UFW) — 공식 문서

---
출처: B1-1 (Layer 2.4) · 학습일: 2026-05-09
