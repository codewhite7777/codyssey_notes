# 포트와 LISTEN 상태

## 개요

TCP/UDP 포트는 IP 주소 위에 추가된 16비트 식별자로, 한 호스트에서 동시에 실행되는 여러 네트워크 서비스를 구분하는 다중화 메커니즘이다. IP 주소가 "어떤 컴퓨터"를 가리킨다면 포트는 "그 컴퓨터의 어떤 서비스"를 가리킨다고 볼 수 있다. 한 호스트가 SSH 데몬(22), HTTP 서버(80), 데이터베이스(5432)를 동시에 운영할 수 있는 이유가 이 분리 덕분이다.

LISTEN은 TCP 상태 머신의 한 상태로, 서버 소켓이 클라이언트 연결을 기다리는 단계를 의미한다. monitor.sh가 "agent_app이 정상 동작 중인가"를 확인할 때 결국 보는 것이 "프로세스가 PID로 살아있는가" + "포트 15034가 LISTEN 상태인가"의 두 신호이며, 후자가 이 노트의 주제다.

## 왜 알아야 하나

운영 디버깅에서 포트 상태 확인은 가장 자주 쓰는 1차 진단이다. "서비스가 안 응답한다"는 보고를 받으면 가장 먼저 (1) 프로세스가 살아있는가, (2) 그 프로세스가 의도한 포트에 LISTEN하고 있는가, (3) 방화벽이 그 포트를 막고 있지는 않은가의 순서로 점검한다. 이 흐름의 절반이 포트와 LISTEN 상태에 대한 이해다.

또한 보안 감사 측면에서도 포트 매핑은 핵심이다. "이 서버가 어떤 서비스를 외부에 노출하고 있는가"를 답하려면 LISTEN 중인 모든 포트를 나열하고 각각이 어떤 프로세스에 연결되는지 매핑해야 한다. `nmap`이 외부에서 보는 시점이라면 `ss -tulnp`는 내부에서 보는 진실이다.

## TCP 상태 머신과 LISTEN

TCP 연결은 단순히 "연결됨/끊김"의 이분법이 아니라 11개 상태를 갖는 유한 상태 머신이다. 운영에서 만나는 주요 상태는 다음과 같다.

```
서버 측 (수동 수립)              클라이언트 측 (능동 수립)

LISTEN                           CLOSED
  │                                │
  │ ← SYN 수신                    │ SYN 송신 →
  ▼                                ▼
SYN_RECV                         SYN_SENT
  │                                │
  │ SYN+ACK 송신 →                │ ← SYN+ACK 수신
  │                                │ ACK 송신 →
  ▼                                ▼
ESTABLISHED                      ESTABLISHED
  ↑                                │
  │ (양방향 데이터 교환)           │
  ▼                                ▼
CLOSE_WAIT                       FIN_WAIT_1
LAST_ACK                         FIN_WAIT_2
                                 TIME_WAIT  ← 종료 후 2MSL 대기
                                 CLOSED
```

`LISTEN`은 서버가 클라이언트의 SYN을 기다리는 상태다. 이 상태가 보이지 않으면 그 포트로 들어오는 연결은 모두 거부된다(RST 응답). monitor.sh의 health check가 "포트가 LISTEN인가"를 검증하는 이유가 이것이다.

`ESTABLISHED`는 양방향 데이터 교환이 가능한 정상 상태다. 활동 중인 연결의 대부분이 여기 있다.

`TIME_WAIT`는 연결을 능동 종료한 쪽이 2MSL(보통 60초~4분) 동안 머무는 상태로, 늦게 도착하는 패킷이 새 연결에 잘못 적용되는 것을 방지한다. 부하 높은 서버에서 TIME_WAIT가 누적되어 포트 고갈로 이어지는 사고가 흔하다.

## 포트 번호 체계

16비트 포트 공간(0~65535)은 역사적·관례적으로 세 영역으로 나뉜다.

```
0-1023        Well-Known Ports (System Ports)
              IANA가 표준 서비스에 할당. SSH 22, HTTP 80, HTTPS 443.
              바인딩에 root 권한 또는 CAP_NET_BIND_SERVICE 필요.

1024-49151    Registered Ports (User Ports)
              IANA에 등록되었거나 관례적으로 사용. PostgreSQL 5432,
              Redis 6379, agent-app의 15034 등.
              일반 사용자도 바인딩 가능.

49152-65535   Dynamic / Ephemeral Ports
              클라이언트가 연결 시 임시로 받는 포트.
              `/proc/sys/net/ipv4/ip_local_port_range`로 범위 확인.
```

이번 과제의 SSH 변경 포트 20022와 앱 포트 15034는 모두 Registered 영역에 속한다. 1024 미만으로 가지 않는 것은 일반 사용자 권한으로 바인딩 가능하게 하기 위한 의도적 선택이다.

## 바인딩 주소: 0.0.0.0 vs 127.0.0.1 vs ::

서비스가 LISTEN할 때 어떤 네트워크 인터페이스에서 받을지를 결정하는 것이 바인딩 주소다. 같은 포트라도 어디에 바인딩되었는지에 따라 도달 가능성이 완전히 달라진다.

`0.0.0.0:15034`는 모든 IPv4 인터페이스에서 받는다는 의미다. 외부에서도, 로컬에서도 접근 가능하므로 일반적으로 외부 서비스용이다. 이번 과제 명세의 "0.0.0.0:15034로 LISTEN"이 이 형태를 요구한다.

`127.0.0.1:15034`(또는 `localhost:15034`)는 같은 호스트 안에서만 접근 가능하다. 내부 컴포넌트 간 통신용으로, 외부 노출이 위험한 관리 인터페이스(예: PostgreSQL의 admin port) 등에 쓴다.

`:::15034` (또는 `[::]:15034`)는 모든 IPv6 인터페이스다. 현대 Linux는 보통 dual-stack으로 설정되어 있어, IPv6 바인딩이 IPv4 트래픽도 함께 받는 경우가 있다(`net.ipv6.bindv6only=0`).

`192.168.1.10:15034`처럼 특정 IP에 바인딩하면 그 인터페이스에서만 받는다. 멀티 홈 서버에서 내부망에만 노출하고 싶을 때 사용한다.

## 한 번 보자

포트 상태를 확인하는 도구로 가장 자주 쓰는 것은 `ss`다. `netstat`의 후계자로 더 빠르고 더 자세하며, 거의 모든 현대 배포판에 기본 설치되어 있다. `-tulnp`가 운영 표준 옵션 조합으로, t(TCP) u(UDP) l(LISTEN만) n(이름 해석 X, 빠름) p(프로세스 정보 포함)을 의미한다.

```bash
sudo ss -tulnp                    # 가장 자주 쓰는 형태
sudo ss -tulnp | grep :15034      # 특정 포트만
sudo ss -tn state established     # ESTABLISHED 상태인 TCP만
sudo ss -tn state time-wait | wc -l   # TIME_WAIT 개수 (부하 진단)

# 출력 예
# Netid State  Recv-Q Send-Q  Local Address:Port  Peer Address:Port
# tcp   LISTEN 0      128     0.0.0.0:15034       0.0.0.0:*    users:(("python",pid=1234,fd=3))
```

`p` 옵션이 핵심인데, 어떤 프로세스가 그 포트를 점유하는지 명확히 보여준다 — `users:(("python",pid=1234,fd=3))`는 PID 1234의 python 프로세스가 fd 3에서 LISTEN 중이라는 의미다. 다만 다른 사용자의 프로세스 정보를 보려면 root 권한이 필요해 `sudo`가 필수다.

`lsof`는 더 일반적인 도구로, 파일 디스크립터·네트워크 연결을 모두 다룬다. 포트 점유 프로세스 디버깅에 강력하다.

```bash
sudo lsof -iTCP:15034 -sTCP:LISTEN    # 15034 LISTEN 중인 프로세스
sudo lsof -i :15034                   # 15034 관련 모든 활동
sudo lsof -p 1234                     # 특정 PID가 연 모든 파일·소켓
```

특정 포트가 누구 차지인지 빠르게 보려면 `fuser`도 유용하다.

```bash
sudo fuser 15034/tcp                  # 그 포트 점유 PID
sudo fuser -k 15034/tcp               # 점유 프로세스 SIGTERM (조심!)
```

연결 상태별 통계가 필요할 땐 `ss -s`가 한눈에 정리해 준다.

```bash
ss -s
# 출력: TCP: total / estab / closed / orphaned / timewait ...
```

## 흔한 함정

포트와 LISTEN 운영에서 만나는 함정은 바인딩 의미·상태 머신·도구의 한계에 분포한다. 가장 흔한 첫 함정은 바인딩 주소를 잘못 설정하는 것이다. 개발 중에는 `127.0.0.1`로 바인딩되어 잘 동작하다가 production에서 외부에서 접근이 안 되는 케이스가 자주 일어나는데, 외부 노출이 필요하면 `0.0.0.0`(IPv4)이나 `::`(IPv6)으로 바인딩해야 한다. 반대로 보안 민감 서비스를 실수로 `0.0.0.0`에 두어 인터넷에 노출하는 사고도 같은 종류의 실수다.

부하 높은 서버에서 자주 만나는 TIME_WAIT 함정도 알아둘 가치가 있다. TCP 연결을 능동 종료한 쪽은 2MSL 동안 TIME_WAIT 상태에 머무는데, 짧은 연결을 대량으로 만드는 패턴(예: 매 요청마다 새 연결)에서 ephemeral 포트가 고갈되어 새 연결을 못 만드는 사고가 발생한다. `net.ipv4.tcp_tw_reuse=1` 같은 sysctl 조정이나, 더 근본적으로는 connection pooling으로 해결한다.

도구의 한계도 함정이 된다. `netstat`은 deprecated이고 일부 새 정보를 못 보여주므로 `ss`로 옮기는 게 표준이지만, 오래된 문서나 시스템에서 여전히 만난다. 또 `ss`의 `-p`는 sudo 없이는 다른 사용자 프로세스를 보여주지 않는데, "프로세스 컬럼이 비어있다"고 의아해하다 권한 부족이 원인이라는 걸 뒤늦게 깨닫는 경우가 흔하다.

IPv4/IPv6 dual-stack 함정도 미묘하다. 현대 Linux는 보통 IPv6 소켓이 IPv4 트래픽도 함께 받도록 구성(`v4-mapped IPv6 addresses`)되어 있어서, `:::15034`로만 보여도 IPv4로 접근 가능한 경우가 많다. 반면 IPv4·IPv6 양쪽에 명시적으로 바인딩하는 서비스는 두 줄로 보이는데, 익숙하지 않으면 "왜 두 개?"하며 헷갈린다. `net.ipv6.bindv6only=1` 설정은 dual-stack을 끄는데, 컨테이너 환경에서 종종 만난다.

마지막으로 컨테이너 환경에서는 포트 LISTEN의 의미가 미묘하게 다르다. 컨테이너 내부에서 LISTEN 중이어도 호스트 외부에서 접근하려면 docker의 `-p` 포트 매핑이 필요하다. `ss -tulnp`가 호스트에서 컨테이너 내부 LISTEN을 보지 못하는 경우가 많아, 컨테이너 내부에서 직접 확인하거나 docker port 명령을 써야 한다.

## B1-1 매핑

이번 과제에서 LISTEN 검증이 필요한 포트는 두 개다 — SSH의 20022, agent-app의 15034.

```bash
# verify.sh 권장 패턴
expected_ports=(20022 15034)
for port in "${expected_ports[@]}"; do
    if sudo ss -tulnp | grep -q ":$port "; then
        echo "[OK] port $port is LISTENing"
    else
        echo "[FAIL] port $port not in LISTEN state"
        exit 1
    fi
done
```

명세는 특히 agent-app이 `0.0.0.0:15034`로 LISTEN한다는 점을 강조한다. 이는 외부 인터페이스에서도 받을 수 있어야 한다는 의도이며, 만약 `127.0.0.1:15034`로만 LISTEN한다면 명세 의도와 다르다. 검증할 때 grep 패턴을 더 엄격히 잡아 `0.0.0.0:15034`나 `*:15034`를 명시적으로 확인하는 게 안전하다.

monitor.sh의 health check에서도 이 검증이 핵심이다. 프로세스가 살아있어도 포트 LISTEN이 깨졌다면 외부 클라이언트가 접근 불가능한 상태이므로, 두 신호를 함께 확인해야 진정한 health check가 된다.

## 인접 토픽

포트와 소켓의 응용은 성능·관측·격리의 세 갈래로 정리해 볼 수 있다.

성능 측면에서는 epoll(Linux)·kqueue(BSD/macOS) 같은 IO multiplexing 메커니즘이 핵심이다. 단일 스레드로 수만 개의 동시 소켓을 처리하는 nginx·redis·node.js의 기반이며, "어떤 소켓이 읽기·쓰기 가능한 이벤트를 가졌나"를 효율적으로 알려준다. select(2)·poll(2)의 한계를 해결한 후속 메커니즘으로, 운영자라면 max open files (`ulimit -n`)와 함께 알아둘 가치가 있다.

소켓 옵션 측면에서는 `SO_REUSEADDR`, `SO_REUSEPORT`, `TCP_NODELAY`, `TCP_KEEPALIVE` 등이 운영 튜닝의 영역이다. 특히 `SO_REUSEPORT`는 같은 포트에 여러 프로세스가 LISTEN할 수 있게 해서, 커널이 들어오는 연결을 자동으로 분배한다. nginx의 worker 프로세스나 멀티 프로세스 서버의 부하 분산 패턴에 활용된다.

관측 측면에서는 eBPF 기반 도구가 새로운 표준으로 자리잡았다. `tcplife`, `tcpconnect`, `tcpaccept` 같은 bcc/bpftrace 도구로 모든 TCP 연결의 lifecycle을 매우 낮은 오버헤드로 추적할 수 있다. production 디버깅에서 `tcpdump`보다 가벼운 대안으로 자주 쓴다.

격리 측면에서는 network namespace가 컨테이너의 기반이다. 각 namespace가 자기만의 네트워크 스택(인터페이스, 라우팅 테이블, iptables 룰, 포트 공간)을 가지므로, 같은 호스트에 같은 포트를 사용하는 여러 컨테이너가 격리되어 동시 실행될 수 있다. `docker run -p 80:80`이 호스트의 80을 컨테이너의 80에 매핑하는 동작도 namespace 간 NAT의 응용이다. `ip netns` 명령으로 namespace를 직접 다룰 수 있다.

마지막으로 클라우드 환경의 보안 그룹(AWS Security Group, GCP Firewall Rule)은 호스트 외부의 네트워크 layer 방화벽이다. 호스트 내부의 ufw·firewalld와 별개로 작동하므로, 클라우드에서는 두 layer 모두 알맞게 설정해야 트래픽이 흐른다. 이 차이를 모르고 호스트 방화벽만 열고 "왜 안 되지" 하는 사고가 흔하다.

## 참고

- `man ss` — 가장 자주 쓰는 포트 도구
- `man tcp` — TCP 프로토콜과 상태 머신
- `man socket` — 소켓 시스템 콜
- `/proc/net/tcp`, `/proc/net/tcp6` — raw 소켓 정보 (ss의 데이터 소스)
- [Linux Network Stack Tuning](https://www.kernel.org/doc/Documentation/networking/) — 커널 문서

---
출처: B1-1 (Layer 2.3) · 학습일: 2026-05-09
