# SPEC-MULTI-002: 멀티패널 레이아웃 및 키보드 네비게이션

## 메타데이터

| 항목 | 값 |
|------|-----|
| SPEC ID | SPEC-MULTI-002 |
| 제목 | 멀티패널 레이아웃 및 키보드 네비게이션 |
| 상태 | Planned |
| 우선순위 | High |
| 생성일 | 2026-03-18 |
| 관련 SPEC | SPEC-MULTI-001 (선행 필수) |

---

## Environment (환경)

### 기술 환경

- SPEC-MULTI-001의 코어 엔진 위에 구축
- 추가 의존성 없음 (ratatui의 Layout 시스템 활용)

### 참고 구현

- **zellij**: 패널 분할 및 탭 시스템, 키보드 모드 전환 참고
- **tmux**: 패널 생성/제거/리사이즈 워크플로우 참고
- **vim**: 모드 기반 입력 처리 패턴 참고

---

## Assumptions (가정)

### 기술 가정

1. ratatui의 Layout::split이 동적 패널 분할에 충분한 유연성을 제공한다
   - 신뢰도: 높음
   - 검증 방법: 2x2, 3열 등 다양한 레이아웃 테스트

2. 동시에 8개 셸을 실행해도 성능 저하가 없다
   - 신뢰도: 중간 (셸 당 비동기 I/O 스레드 필요)
   - 검증 방법: 8개 셸 동시 출력 시 CPU/메모리 모니터링

3. 모드 기반 입력(Normal/Insert)이 사용자 학습 곡선을 낮출 수 있다
   - 신뢰도: 중간
   - 위험: vim 스타일에 익숙하지 않은 사용자에게 혼란 가능

---

## Requirements (요구사항)

### 유비쿼터스 요구사항

- **[REQ-101]** 시스템은 **항상** 최소 2개, 최대 8개의 셸 패널을 동시에 표시할 수 있어야 한다
- **[REQ-102]** 시스템은 **항상** 하나의 패널에 시각적 포커스 표시(테두리 색상)를 해야 한다
- **[REQ-103]** 시스템은 **항상** 상태바에 현재 모드, 활성 셸 정보, 단축키 안내를 표시해야 한다

### 이벤트 기반 요구사항

- **[REQ-110]** **WHEN** 사용자가 Normal 모드에서 셸 생성 단축키를 누르면 **THEN** 새 셸 패널이 생성되고 레이아웃이 재계산되어야 한다
- **[REQ-111]** **WHEN** 사용자가 Normal 모드에서 방향 단축키(h/j/k/l 또는 화살표)를 누르면 **THEN** 해당 방향의 패널으로 포커스가 이동해야 한다
- **[REQ-112]** **WHEN** 사용자가 Normal 모드에서 셸 종료 단축키를 누르면 **THEN** 활성 셸이 종료되고 레이아웃이 재계산되어야 한다
- **[REQ-113]** **WHEN** 사용자가 Normal 모드에서 Enter 또는 i를 누르면 **THEN** Insert 모드로 전환되어 키 입력이 셸로 전달되어야 한다
- **[REQ-114]** **WHEN** 사용자가 Insert 모드에서 Escape를 누르면 **THEN** Normal 모드로 전환되어야 한다
- **[REQ-115]** **WHEN** 패널 수가 변경되면 **THEN** 0.5초 이내에 새 레이아웃으로 전환되어야 한다

### 상태 기반 요구사항

- **[REQ-120]** **IF** Normal 모드 상태이면 **THEN** 모든 키 입력을 단축키로 해석해야 한다
- **[REQ-121]** **IF** Insert 모드 상태이면 **THEN** Escape 키를 제외한 모든 키 입력을 활성 셸로 전달해야 한다
- **[REQ-122]** **IF** 셸이 1개만 남은 상태에서 해당 셸을 종료하면 **THEN** 애플리케이션을 종료해야 한다

### 금지 요구사항

- **[REQ-130]** Insert 모드에서 셸 관리 단축키가 동작**하지 않아야 한다** (Escape 제외)

---

## Specifications (사양)

### 레이아웃 알고리즘

패널 수에 따른 자동 레이아웃:

| 패널 수 | 레이아웃 |
|---------|---------|
| 1 | 전체 화면 |
| 2 | 좌우 분할 (50:50) |
| 3 | 좌 50% + 우 상하 분할 |
| 4 | 2x2 격자 |
| 5-6 | 2행 x 3열 (빈 칸 허용) |
| 7-8 | 2행 x 4열 (빈 칸 허용) |

### 입력 모드 설계

```
Normal Mode (기본)
├── h/j/k/l 또는 방향키: 패널 포커스 이동
├── n: 새 셸 생성
├── x: 활성 셸 종료
├── Enter 또는 i: Insert 모드 전환
├── 1-8: 해당 번호 패널로 직접 이동
└── Ctrl+Q: 앱 종료

Insert Mode (셸 입력)
├── Escape: Normal 모드 복귀
└── 모든 키: 활성 셸 PTY로 전달
```

### 추가 모듈

```
src/ui/
├── layout.rs           # LayoutManager (패널 수 기반 레이아웃 계산)
└── statusbar.rs        # StatusBar (모드, 셸 정보, 단축키 표시)
src/input/
└── keybindings.rs      # KeyBindings (단축키 매핑 정의)
```

### 상태바 정보

```
[NORMAL] Shell 2/4 | PowerShell | n:New x:Close i:Insert Ctrl+Q:Quit
[INSERT] Shell 2/4 | PowerShell | Esc:Normal
```

---

## Traceability (추적성)

| 태그 | 관련 파일 |
|------|-----------|
| SPEC-MULTI-002 | spec.md, plan.md, acceptance.md |
| REQ-101 ~ REQ-103 | src/ui/layout.rs, src/ui/statusbar.rs |
| REQ-110 ~ REQ-115 | src/shell/manager.rs, src/ui/layout.rs |
| REQ-120 ~ REQ-122 | src/input/handler.rs, src/input/keybindings.rs |
| REQ-130 | src/input/handler.rs |
