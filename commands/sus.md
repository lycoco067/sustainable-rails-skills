# /sus - Sustainable Rails Commander

"Sustainable Web Development with Rails" 책 기반의 중앙 라우팅 에이전트.
요청을 분석하고 적절한 하위 에이전트로 위임합니다.

## Trigger Keywords
- sus, sustainable rails
- rails best practices
- 지속가능한 레일즈

---

## Routing Matrix

| Request Pattern | Target Agent | Description |
|----------------|--------------|-------------|
| setup, init, bin/setup | `/sus-setup` | 프로젝트 초기 설정 |
| implement, build, create, feature | `/sus-implement` | 기능 구현 |
| test, coverage, spec | `/sus-test` | 테스트 작성 |
| ops, deploy, auth, api, logging | `/sus-ops` | 운영 관련 |
| check, audit, compliance | `/sustainable-check` | 원칙 준수 검사 |
| refactor, service, fat model | `/refactor-to-services` | 서비스 리팩토링 |

---

## Command Variants

```bash
/sus                      # 인터랙티브 모드 - 무엇이 필요한지 물어봄
/sus check                # 원칙 준수 검사 실행
/sus setup                # 프로젝트 설정 점검
/sus implement <feature>  # 기능 구현 가이드
/sus test                 # 테스트 전략 및 작성
/sus ops                  # 운영 설정 가이드
/sus status               # 현재 프로젝트 상태 요약
```

---

## Workflow

### Phase 1: Context Assessment
```yaml
analyze:
  - current_branch: git branch --show-current
  - changed_files: git status --porcelain
  - recent_work: 최근 작업 파일 분석
```

### Phase 2: Request Classification
```
사용자 요청
    │
    ├─ 설정 관련 키워드 → /sus-setup
    ├─ 구현 관련 키워드 → /sus-implement
    ├─ 테스트 관련 키워드 → /sus-test
    ├─ 운영 관련 키워드 → /sus-ops
    ├─ 검사 관련 키워드 → /sustainable-check
    └─ 리팩토링 키워드 → /refactor-to-services
```

### Phase 3: Delegation
선택된 에이전트 호출 후:
1. 에이전트 실행
2. 결과 품질 검사 (선택적)
3. 추가 작업 제안

---

## Interactive Mode

`/sus` 단독 실행 시:

```
═══════════════════════════════════════════════════════════════
🛤️ Sustainable Rails Commander
═══════════════════════════════════════════════════════════════

무엇을 도와드릴까요?

1. 🔧 프로젝트 설정 점검 (/sus setup)
2. 🏗️ 새 기능 구현 (/sus implement)
3. 🧪 테스트 작성 (/sus test)
4. 🚀 운영 설정 (/sus ops)
5. ✅ 원칙 준수 검사 (/sus check)

숫자 또는 키워드로 선택하세요:
```

---

## Status Command

`/sus status` 실행 시:

```
═══════════════════════════════════════════════════════════════
📊 Sustainable Rails Status Report
═══════════════════════════════════════════════════════════════

프로젝트: [Project Name]
브랜치: [current branch]

📁 설정 상태:
   ✅ bin/setup - 정상
   ✅ bin/ci - 정상
   ✅ lograge - 설정됨

📋 최근 검사 결과: (없음 - /sus check 실행 권장)

🔧 권장 작업:
   1. /sus check로 전체 검사 실행
   2. 새 기능 구현 시 /sus implement 사용
```

---

## Quality Gate (Post-Completion)

하위 에이전트 작업 완료 후 자동 실행:

```yaml
quality_checks:
  routes:
    - resources 사용 여부
    - :only 명시 여부
  views:
    - 단일 인스턴스 변수
    - 파셜 locals 사용
  helpers:
    - content_tag 사용
    - html_safe 미사용
```

위반 발견 시:
```
⚠️ Quality Gate Warning:
   - app/views/widgets/_card.html.erb:5 - @widget 직접 사용 (locals 권장)

   자동 수정하시겠습니까? (y/n)
```

---

## Sub-Agent Integration

### 에이전트 연동
```
/sus check       → sustainable-check.md
/sus refactor    → sub_made/refactor-to-services.md
```

### 구현 에이전트
```
/sus setup       → sus-setup.md
/sus implement   → sus-implement.md
/sus test        → sus-test.md
/sus ops         → sus-ops.md
```

---

## Examples

### 새 기능 구현
```
User: /sus implement 리더보드 기능
Commander: 🏗️ /sus-implement로 라우팅합니다...

[sus-implement 에이전트 실행]
1. Routes 설계
2. Service 클래스 생성
3. Controller 구현
4. Views 작성
5. Tests 작성 (/sus-test 연동)
```

### 프로젝트 점검
```
User: /sus check
Commander: ✅ /sustainable-check로 라우팅합니다...

[sustainable-check 에이전트 실행]
Routes: 28/30
Templates: 32/40
Helpers: 22/30
Total: 82/100 ⭐⭐⭐⭐
```
