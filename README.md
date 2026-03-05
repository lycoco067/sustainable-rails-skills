# Sustainable Rails Skills for Claude Code

"Sustainable Web Development with Rails" 책 기반의 Claude Code 스킬 모음입니다.
Rails 프로젝트에서 지속 가능한 코드 품질과 베스트 프랙티스를 유지하도록 도와줍니다.

## 설치 방법

### 1. 파일 복사

프로젝트의 `.claude/commands/` 디렉토리에 복사합니다:

```bash
# 프로젝트 루트에서
mkdir -p .claude/commands/sus

# 이 저장소를 클론하거나 다운로드한 후
cp -r commands/* .claude/commands/sus/
```

### 2. 디렉토리 구조

설치 후 구조:
```
.claude/
└── commands/
    └── sus/
        ├── sus.md                    # 메인 커맨더 (라우팅)
        ├── sus-setup.md              # 프로젝트 설정
        ├── sus-implement.md          # 기능 구현 오케스트레이터
        ├── sus-test.md               # 테스팅 에이전트
        ├── sus-ops.md                # 운영 설정
        ├── sustainable-check.md      # 원칙 준수 검사
        ├── impl/                     # 구현 원칙 서브에이전트
        │   ├── routes.md
        │   ├── models.md
        │   ├── services.md
        │   ├── controllers.md
        │   ├── views.md
        │   ├── helpers.md
        │   ├── frontend.md
        │   ├── jobs.md
        │   ├── mailers.md
        │   └── rake_tasks.md
        └── sub_made/
            └── refactor-to-services.md
```

## 사용 방법

### 메인 커맨더

```bash
/sus                      # 인터랙티브 모드
/sus check                # 원칙 준수 검사
/sus setup                # 프로젝트 설정 점검
/sus implement <feature>  # 기능 구현 가이드
/sus test                 # 테스트 전략 및 작성
/sus ops                  # 운영 설정 가이드
/sus status               # 현재 프로젝트 상태 요약
```

### 직접 호출

```bash
/sus-setup                # 프로젝트 설정 검사
/sus-implement --backend  # 백엔드 구현 (models + services + controllers)
/sus-test system          # System test 템플릿 생성
/sus-ops auth             # 인증 설정 체크리스트
/sustainable-check        # 전체 원칙 준수 검사
```

## 스킬 개요

### `/sus` - Sustainable Rails Commander
메인 라우팅 에이전트. 요청을 분석하고 적절한 하위 에이전트로 위임합니다.

### `/sus-setup` - Project Setup Agent
- bin/ 스크립트 점검 (bin/setup, bin/ci, bin/dev)
- 환경 변수 설정 (.env 파일)
- lograge 로깅 설정
- CI/CD 파이프라인 설정

### `/sus-implement` - Implementation Orchestrator
기능 구현 시 필요한 레이어를 분석하고 서브에이전트를 **병렬로** 실행합니다.

| 플래그 | 서브에이전트 |
|--------|-------------|
| `--routes` | impl/routes.md |
| `--backend` | models + services + controllers |
| `--frontend` | frontend + views + helpers |
| `--model` | impl/models.md |
| `--service` | impl/services.md |

### `/sus-test` - Testing Agent
Testing Pyramid 기반의 테스트 전략:

```
System Tests (Few, high-value)
    ↓
Integration Tests (Moderate)
    ↓
Unit Tests (Many, fast)
    ↓
JavaScript Tests (Critical paths)
```

**주요 명령어:**
- `/sus-test system` - System test 템플릿
- `/sus-test service <name>` - Service test 생성
- `/sus-test scenario` - 시나리오 병렬 실행 + 오류 자동 수정
- `/sus-test journey` - 유저 저니 스크린샷 캡쳐

### `/sus-ops` - Operations Agent
- 인증 (Devise, OAuth, 권한 부여)
- API 설계 (Resource-oriented, Versioning)
- 로깅 (4대 기법: Request ID, Identifiers, Source, User ID)
- 예외 처리 (Sentry)
- 모니터링 (비즈니스 결과 중심)

### `/sustainable-check` - Principle Checker
코드 품질 검사 및 점수 산출:

| 영역 | 가중치 |
|------|--------|
| Routes | 30% |
| Templates | 40% |
| Helpers | 30% |

## 핵심 원칙

### Routes
- `resources` with `:only` 필수
- 커스텀 액션 금지 → 새 리소스로 분리
- 중첩 1단계 이하

### Models
- Active Record = DB 접근만
- 비즈니스 로직 → Services
- DB Constraints for 데이터 무결성

### Services
- Stateless 클래스
- Rich Result 객체 반환
- Single responsibility

### Views
- 컨트롤러당 하나의 인스턴스 변수
- 파셜은 `locals:` 필수
- 시맨틱 HTML

### Testing
- data-testid 패턴
- System tests는 주요 플로우만
- Vitest for JavaScript

## 프로젝트 커스터마이징

### ApplicationService 패턴

스킬은 다음과 같은 서비스 패턴을 가정합니다:

```ruby
# app/services/application_service.rb
class ApplicationService
  def self.call(**args)
    new(**args).call
  end

  private

  def success(value = {})
    Result.new(success: true, value: value)
  end

  def failure(error)
    Result.new(success: false, error: error)
  end

  Result = Struct.new(:success, :value, :error, keyword_init: true) do
    def success?
      success
    end

    def failure?
      !success
    end
  end
end
```

### 프로젝트별 설정

각 `.md` 파일의 "프로젝트 특화" 섹션을 수정하여 프로젝트에 맞게 커스터마이징하세요.

## 참고 자료

- [Sustainable Web Development with Rails](https://sustainable-rails.com/) by David Bryant Copeland
- [Rails 8 Documentation](https://guides.rubyonrails.org/)

## 라이선스

MIT License
