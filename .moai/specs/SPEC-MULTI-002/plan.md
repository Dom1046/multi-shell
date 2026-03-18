# SPEC-MULTI-002: 구현 계획

## 관련 SPEC

- SPEC ID: SPEC-MULTI-002
- 제목: 멀티패널 레이아웃 및 키보드 네비게이션
- 선행 SPEC: SPEC-MULTI-001

---

## 마일스톤

### Primary Goal: 입력 모드 시스템 구현

- Normal/Insert 이중 모드 상태 머신 구현 (src/input/handler.rs)
- 단축키 매핑 정의 (src/input/keybindings.rs)
- 모드 전환 로직 (Enter/i -> Insert, Escape -> Normal)
- App 구조체에 InputMode enum 추가
- 난이도: 중간

### Secondary Goal: 멀티 셸 관리

- ShellManager를 다중 셸 지원으로 확장
- 셸 생성(n키) 및 종료(x키) 명령 처리
- 활성 셸 인덱스 관리 및 포커스 전환
- 셸 종료 시 나머지 셸 재인덱싱
- 난이도: 중간

### Tertiary Goal: 동적 레이아웃 엔진

- LayoutManager 구현 (src/ui/layout.rs)
- 패널 수에 따른 자동 레이아웃 계산 (1~8개)
- ratatui Layout::split을 활용한 동적 분할
- 포커스 패널 시각적 강조 (테두리 색상 변경)
- 터미널 리사이즈 대응
- 난이도: 어려움 (다양한 패널 조합의 레이아웃 계산)

### Final Goal: 상태바 및 통합

- StatusBar 위젯 구현 (src/ui/statusbar.rs)
- 현재 모드, 셸 번호/총수, 셸 제목 표시
- 모드별 단축키 안내 텍스트
- 전체 통합 테스트 및 안정화
- 난이도: 쉬움

---

## 기술 접근법

### 입력 모드 상태 머신

```
enum InputMode {
    Normal,  // 단축키 모드 (패널 네비게이션, 셸 관리)
    Insert,  // 셸 입력 모드 (모든 키를 PTY로 전달)
}
```

- Normal 모드가 기본 상태
- 모드 전환은 즉각적이며 상태바에 즉시 반영
- Insert 모드에서 Escape는 항상 Normal로 복귀 (셸로 전달하지 않음)

### 레이아웃 계산 전략

ratatui의 `Layout`과 `Constraint`를 활용한 선언적 레이아웃:

1. 전체 영역을 상태바(1줄) + 메인 영역으로 수직 분할
2. 메인 영역을 패널 수에 따라 행/열로 분할
3. 각 셀에 패널 또는 빈 공간 배치

재계산 트리거:
- 셸 생성/종료 시
- 터미널 리사이즈 시
- 레이아웃 프리셋 변경 시 (Post-MVP)

### 포커스 관리

- `active_shell` 인덱스로 포커스 추적
- 방향키/hjkl로 논리적 방향 이동 (그리드 기반)
- 숫자키(1-8)로 직접 패널 선택
- 포커스 변경 시 테두리 색상 업데이트 (강조: Cyan, 비활성: DarkGray)

---

## 아키텍처 설계 방향

### 모듈 확장

SPEC-MULTI-001의 구조를 확장:

```
src/input/
├── handler.rs      # InputHandler (모드 기반 키 분류 확장)
└── keybindings.rs  # KeyBindings (단축키 정의, 향후 사용자 정의 가능)

src/ui/
├── panel.rs        # Panel (다중 패널 렌더링 확장)
├── layout.rs       # LayoutManager (NEW)
└── statusbar.rs    # StatusBar (NEW)
```

### 셸 관리 확장

ShellManager의 주요 변경:
- `Vec<ShellInstance>` 관리
- 셸 추가/제거 시 안전한 인덱스 관리
- 각 셸의 독립적 출력 버퍼 유지
- 셸 종료 이벤트 감지 및 알림

---

## 리스크 및 대응 계획

| 리스크 | 발생 가능성 | 영향도 | 대응 계획 |
|--------|------------|--------|-----------|
| 복잡한 레이아웃에서 렌더링 깨짐 | 중간 | 중간 | 패널 최소 크기 강제, 단위 테스트 |
| 8개 셸 동시 실행 시 성능 저하 | 중간 | 중간 | 비활성 패널 렌더링 빈도 낮춤 |
| 모드 전환 혼란 | 높음 | 낮음 | 상태바에 모드 명확히 표시, 색상 구분 |
| 포커스 이동 방향 직관성 부족 | 중간 | 낮음 | 그리드 기반 논리적 방향 이동 |

---

## 파일 구조 및 모듈 분할 계획

| 파일 | 예상 라인 수 | 핵심 책임 |
|------|-------------|-----------|
| src/ui/layout.rs | 80-120 | 패널 수 기반 레이아웃 계산, Constraint 생성 |
| src/ui/statusbar.rs | 40-60 | 모드/셸 정보/단축키 표시 위젯 |
| src/ui/panel.rs (수정) | +30-50 | 다중 패널 렌더링, 포커스 표시 |
| src/input/keybindings.rs | 40-60 | 단축키 enum, 매핑 테이블 |
| src/input/handler.rs (수정) | +50-80 | 모드 기반 입력 분류, Normal 모드 명령 |
| src/shell/manager.rs (수정) | +40-60 | 다중 셸 관리, 인덱스 관리 |
| src/app.rs (수정) | +30-50 | InputMode 상태, 모드 전환 |

---

## 테스트 전략 개요

### 단위 테스트

- LayoutManager: 패널 1~8개에 대한 레이아웃 결과 검증
- InputHandler: Normal/Insert 모드별 키 라우팅 검증
- KeyBindings: 단축키 매핑 정확성 검증
- ShellManager: 다중 셸 생성/종료/인덱스 관리 검증

### 통합 테스트

- 셸 2개 생성 -> 포커스 전환 -> 각 셸에 명령 입력 -> 출력 확인
- 셸 생성/종료 반복 시 레이아웃 일관성 검증
- 모드 전환 플로우 전체 검증

### 수동 테스트 체크리스트

- 모든 단축키 동작 확인
- 다양한 터미널 크기에서 레이아웃 확인
- 모드 전환 시 상태바 업데이트 확인
