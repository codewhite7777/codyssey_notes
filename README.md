# Codyssey 학습 노트

> [Codyssey](https://www.codyssey.kr/) 부트캠프 과제를 진행하며 누적되는 학습 자산.
> Linux/OS, Python, Git, AI/SW 기초 개념을 정리·기록.

**현재 진행 단계**: B1-1 시스템 관제 자동화 스크립트 개발 (Linux/OS, 40h)

---

## 이 레포는 무엇인가

학생 입장에서 정답을 빠르게 만들기보다 **개념을 폭넓게 이해하고 사고 과정을 형성**하는 데 초점을 둔 학습 노트 모음입니다.

- 과제 코드 자체는 **별도 레포**에 (예: `codyssey_b1_1`)
- 이 레포는 **개념·회고·진행 현황**만 보관 (워크스페이스 패턴)
- Claude Code의 통합 컨텍스트로 과제 간 학습이 누적되도록 구성

## 디렉토리 구조

```
codyssey_notes/
├── INDEX.md                    # 15개 과제 진행 보드
├── CLAUDE.md                   # 학습 우선 원칙 + Claude Code 컨벤션
├── concepts/                   # 분야별 누적 개념 정리
│   ├── linux/                  # 파일시스템·권한·셸·프로세스 등
│   ├── os/                     # 메모리·스케줄링·동시성
│   ├── python/                 # 데이터 모델·제너레이터·데코레이터
│   └── git/                    # 브랜치·PR·충돌 해결
├── retrospectives/             # 주차별·과제별 회고
└── .claude/commands/           # 학습 워크플로우 슬래시 명령
```

## 학습 방식

### 워크스페이스 패턴
- **이 레포** = 노트·개념·회고만 (GitHub 백업 + 동기화)
- **각 과제는 별도 레포** = 제출·격리·포트폴리오 단위
- 부모 디렉토리에서 Claude Code 실행 → 통합 컨텍스트로 모든 과제 작업

### Learning-First 원칙 ([CLAUDE.md](./CLAUDE.md))

1. **코드 작성 전에 개념 먼저** — 정의 / 왜 / 원리 / 예시 / 오해
2. **정답보다 사고 과정** — 즉시 코드 덤프 X, 단계적 힌트
3. **"왜?"** 를 항상 동반 — 모든 결정에 이유
4. **작업 후 개념 추출** — `concepts/<분야>/`에 정리
5. **과제 간 연결 명시** — 새 학습을 기존 노트와 잇기

### 초심자 모드 (Beginner Mode)

해당 분야를 처음 접할 때 적용되는 추가 규칙:
- 일관된 **비유** 사용 (예: 리눅스 = 아파트 단지, 프로세스 = 일하는 직원)
- 기술 용어는 본문 마지막 **"용어 풀이"** 섹션에 분리
- 본문 첫머리에 **"왜 알아야 해?"** 고정
- ASCII 다이어그램으로 시각화
- "한 번 보자" 섹션으로 직접 실행 가능 명령 제공

## 진행 현황

전체 보드는 **[INDEX.md](./INDEX.md)** 참조.

| ID | 과제 | 상태 | 정리된 개념 |
|---|---|---|---|
| B1-1 | 시스템 관제 자동화 스크립트 | 🟡 학습 중 | [linux/](./concepts/linux/) (5개) |
| B1-2 | 프로세스·리소스 트러블슈팅 | 🔵 시작 전 | — |
| B1-3 | 파일 기반 가계부 | 🔵 시작 전 | — |
| B1-4 | Git 협업 워크플로우 | 🔵 시작 전 | — |
| ... | (나머지 11개) | 🔵 | — |

## Claude Code 슬래시 명령

| 명령 | 용도 |
|---|---|
| `/learn <주제>` | 개념을 단계적으로 깊이 설명 |
| `/connect` | 현재 작업과 기존 `concepts/` 노트 연결 찾기 |
| `/extract` | 작업에서 학습 개념 추출 → `concepts/`에 정리 |
| `/start <id> <name>` | 새 과제 레포 부트스트랩 |
| `/recap` | 세션 회고 노트 초안 |

## 정리된 개념 (현재까지)

### Linux Foundation
- [filesystem-tree.md](./concepts/linux/filesystem-tree.md) — 파일시스템 = 아파트 단지
- [users-and-groups.md](./concepts/linux/users-and-groups.md) — 사용자·그룹 = 주민·동호회
- [file-permissions.md](./concepts/linux/file-permissions.md) — 권한 = 출입 표찰
- [shell-environment.md](./concepts/linux/shell-environment.md) — 셸 환경 = 출근 메모
- [process-and-signals.md](./concepts/linux/process-and-signals.md) — 프로세스 = 일하는 직원

전체 분야별 인덱스는 **[concepts/README.md](./concepts/README.md)** 참조.

## 참고

이 레포는 **부트캠프 학습자 1인의 개인 노트**입니다. 정확성을 보장하지 않으며, 같은 학습 단계의 다른 분에게 참고가 되었으면 하는 마음으로 공개합니다.

오류·개선 제안은 issue로 환영합니다.

## 사용 도구

- **Claude Code** (Anthropic) — 학습 페어로 활용
- **GitHub** — 노트 백업·동기화
- **Markdown** — 모든 노트 양식
