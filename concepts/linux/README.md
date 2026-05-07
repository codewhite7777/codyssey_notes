# Linux 개념

## 정리된 노트

### Layer 1 — Foundation (B1-1)
- [filesystem-tree.md](./filesystem-tree.md) — 디렉토리 트리·FHS·`/etc`·`/var`·`/proc`
- [users-and-groups.md](./users-and-groups.md) — UID·GID·primary vs supplementary group
- [file-permissions.md](./file-permissions.md) — rwx·8진법·chmod·umask
- [shell-environment.md](./shell-environment.md) — `.bashrc` vs `.bash_profile`·export·cron 함정
- [process-and-signals.md](./process-and-signals.md) — PID·SIGTERM vs SIGKILL·`ps`·`kill`

### Layer 2 — 보안 & 네트워킹 (예정, B1-1 진행 중 추가)
- ssh-deep-dive.md
- sshd-config.md
- ports-and-listening.md
- firewall-ufw-vs-firewalld.md
- acl.md

### Layer 3 — 자원 관측 (예정)
- proc-filesystem.md
- cpu-measurement.md
- memory-measurement.md
- disk-usage-df-vs-du.md

### Layer 4 — Bash 스크립팅 (예정)
- bash-shebang.md
- bash-exit-code.md
- bash-set-options.md
- bash-trap.md

### Layer 5 — 자동화 & 운영 (예정)
- cron-format.md
- cron-environment.md
- log-rotation.md

## 추가 가이드
- 학습 후 `/extract`로 개념 추출
- 잘못 알았던 부분은 노트의 "흔한 오해" 섹션으로 보강
- 한 노트 = 한 개념 (5분 안에 읽을 분량)
