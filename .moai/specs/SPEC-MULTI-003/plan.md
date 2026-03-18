# SPEC-MULTI-003: 구현 계획

## 관련 SPEC

- SPEC ID: SPEC-MULTI-003
- 제목: 출력 스크롤백 및 설정 관리
- 선행 SPEC: SPEC-MULTI-001, SPEC-MULTI-002

---

## 마일스톤

### Primary Goal: 스크롤백 버퍼 구현

- ScrollbackBuffer 구조체 설계 (VecDeque 기반 링 버퍼)
- ShellInstance에 ScrollbackBuffer 통합
- 스크롤 오프셋 관리 (PageUp/PageDown, Ctrl+U/D)
- 자동 스크롤 / 수동 스크롤 모드 전환 로직
- 패널 렌더링에서 scroll_offset 반영
- 난이도: 중간

### Secondary Goal: 설정 파일 시스템

- Settings 구조체 정의 (serde 직렬화/역직렬화)
- TOML 설정 파일 로드 및 기본값 적용
- dirs 크레이트로 크로스 플랫폼 설정 경로 결정
- 잘못된 설정값 검증 및 폴백 처리
- 난이도: 쉬움

### Tertiary Goal: CLI 인자 파싱

- clap으로 CLI 인자 정의
- 설정 파일 값과 CLI 인자 병합 (CLI 우선)
- --config, --shell, --num, --scrollback, --verbose 등
- 도움말 및 버전 표시
- 난이도: 쉬움

### Final Goal: 출력 검색 기능

- Search 모드 상태 추가 (InputMode::Search)
- 검색 입력 UI (하단 입력바)
- 텍스트 검색 알고리즘 (단순 문자열 매칭)
- 검색 결과 하이라이트 렌더링
- n/N으로 결과 간 네비게이션
- 난이도: 중간

### Optional Goal: 실시간 검색 (Incremental Search)

- 검색어 입력 시 실시간 하이라이트 업데이트
- 성능 최적화 (대량 버퍼에서의 검색 지연 방지)
- 난이도: 중간

---

## 기술 접근법

### 스크롤백 버퍼 전략

`VecDeque<String>`을 사용한 FIFO 링 버퍼:

- 새 줄 추가: `push_back()`, max_lines 초과 시 `pop_front()`
- 스크롤 오프셋: 0이 맨 아래 (최신), 양수가 위로 스크롤
- 가시 영역 계산: `buffer[len - offset - visible_height .. len - offset]`

메모리 최적화:
- 기본 10,000줄, 줄당 평균 80바이트 가정 시 약 800KB
- 8개 셸 동시: 약 6.4MB (충분히 경량)

### 설정 로드 우선순위

```
1. CLI 인자 (최우선)
2. 환경 변수 (MULTI_SHELL_*)
3. 설정 파일 ({config_dir}/multi-shell/config.toml)
4. 컴파일 시 기본값 (Default trait 구현)
```

### 검색 구현 전략

1단계: 단순 문자열 매칭 (`str::contains`)
- 대소문자 무시 옵션
- 검색 결과 인덱스 캐싱 (재검색 방지)

2단계 (선택): 정규식 검색
- `regex` 크레이트 활용
- 성능 주의 (대량 버퍼에서 정규식 비용)

---

## 아키텍처 설계 방향

### 설정 시스템 설계

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
struct Settings {
    general: GeneralSettings,
    display: DisplaySettings,
    keybindings: KeybindingSettings,
}

impl Default for Settings {
    fn default() -> Self {
        // 합리적인 기본값 제공
    }
}
```

- `serde`로 TOML 직렬화/역직렬화
- `Default` trait으로 기본값 보장
- `clap`의 파싱 결과를 Settings에 병합

### 검색 모드 확장

InputMode를 3개로 확장:

```rust
enum InputMode {
    Normal,     // 단축키 모드
    Insert,     // 셸 입력 모드
    Search,     // 검색 모드 (NEW)
}
```

Search 모드에서:
- 하단에 검색 입력바 렌더링
- 키 입력을 검색어 버퍼로 전달
- Enter로 검색 확정, Escape로 취소

---

## 리스크 및 대응 계획

| 리스크 | 발생 가능성 | 영향도 | 대응 계획 |
|--------|------------|--------|-----------|
| 대량 출력 시 스크롤백 메모리 급증 | 낮음 | 중간 | max_lines 강제 제한, 경고 로그 |
| TOML 파싱 에러로 앱 시작 실패 | 낮음 | 높음 | 파싱 에러 시 기본값 폴백, 경고 표시 |
| 검색 시 대량 버퍼 성능 저하 | 중간 | 낮음 | 검색 결과 캐싱, 비동기 검색 고려 |
| 크로스 플랫폼 설정 경로 불일치 | 낮음 | 낮음 | dirs 크레이트로 추상화 |

---

## 파일 구조 및 모듈 분할 계획

| 파일 | 예상 라인 수 | 핵심 책임 |
|------|-------------|-----------|
| src/config/mod.rs | 10-20 | Config 모듈 re-export |
| src/config/settings.rs | 80-120 | Settings 구조체, TOML 로드, CLI 병합, 기본값 |
| src/shell/manager.rs (수정) | +40-60 | ScrollbackBuffer 통합, 스크롤 관리 |
| src/ui/panel.rs (수정) | +60-80 | 스크롤 렌더링, 검색 하이라이트 |
| src/input/handler.rs (수정) | +30-50 | Search 모드 입력 처리, 스크롤 단축키 |
| src/main.rs (수정) | +20-30 | clap CLI 파싱, Settings 초기화 |

---

## 테스트 전략 개요

### 단위 테스트

- ScrollbackBuffer: 추가/삭제, 최대 크기 제한, 스크롤 오프셋 계산
- Settings: TOML 파싱, 기본값 적용, 잘못된 값 폴백
- 검색: 문자열 매칭, 결과 인덱스, n/N 네비게이션

### 통합 테스트

- 설정 파일 로드 -> 앱 시작 -> 설정 반영 확인
- 대량 출력 -> 스크롤 -> 검색 전체 플로우
- CLI 인자로 설정 오버라이드 검증

### 벤치마크

- 10,000줄 스크롤백 버퍼 메모리 사용량
- 10,000줄에서 검색 응답 시간
- 대량 출력 시 버퍼 추가 성능
