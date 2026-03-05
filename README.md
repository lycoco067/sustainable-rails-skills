# Sustainable Rails Skills for Claude Code

> ["Sustainable Web Development with Rails"](https://sustainable-rails.com/) 책 기반의 Claude Code 스킬 모음

Rails 애플리케이션을 **지속 가능한 개발 원칙**에 따라 구축할 수 있도록 도와주는 Claude Code 명령어 세트입니다. 라우트, 모델, 서비스, 컨트롤러, 뷰, 테스트, 운영까지 베스트 프랙티스를 자동으로 적용합니다.

## 핵심 기능

### `/sus-test` - 자동 테스트 + 오류 수정 + 스크린샷

```bash
/sus-test
```

**이 명령어 하나로:**
1. 테스트 실행
2. 오류 발견 시 자동으로 코드 수정
3. 테스트 통과할 때까지 반복
4. 완료되면 스크린샷 캡처
5. Finder에서 스크린샷 폴더 자동 열기

> 테스트 작성 → 실행 → 디버깅 → 수정의 반복 작업을 **완전 자동화**합니다.

---

## 주요 기능

- **자동 테스트 루프** - 오류 잡을 때까지 자동 실행 + 수정 + 스크린샷
- **원칙 기반 개발** - Sustainable Rails 패턴을 자동으로 적용
- **병렬 실행** - 서브 에이전트가 동시에 실행되어 빠른 구현
- **테스팅 피라미드** - System, Unit, Service, JavaScript 테스트 템플릿
- **서비스 레이어 패턴** - ApplicationService와 Result 객체
- **운영 설정** - 인증, API 설계, 로깅, 모니터링, 에러 처리

## 빠른 시작

```bash
# 저장소 클론
git clone https://github.com/lycoco067/sustainable-rails-skills.git /tmp/sus-skills

# Rails 프로젝트에 복사
mkdir -p .claude/commands
cp -r /tmp/sus-skills/commands .claude/commands/sus

# 정리
rm -rf /tmp/sus-skills
```

## 명령어

### 메인 라우터

```bash
/sus                      # 인터랙티브 모드 - 요청 분석 후 적절한 에이전트로 라우팅
/sus check                # 원칙 준수 검사
/sus setup                # 프로젝트 설정 점검
/sus implement <기능>     # 기능 구현 가이드
/sus test                 # 테스트 전략 및 템플릿
/sus ops                  # 운영 설정 가이드
```

### 직접 호출

| 명령어 | 설명 |
|--------|------|
| `/sus-setup` | bin/ 스크립트, 환경변수, 로깅, CI/CD 점검 |
| `/sus-implement` | 레이어별 구현 오케스트레이션 |
| `/sus-test` | 테스팅 피라미드 기반 테스트 생성 |
| `/sus-ops` | 인증, API, 로깅, 모니터링 설정 |
| `/sustainable-check` | 전체 코드베이스 원칙 검사 |
| `/refactor-to-services` | 비즈니스 로직을 서비스로 추출 |

### 구현 플래그

```bash
/sus-implement <기능>        # 전체 분석 + 병렬 실행
/sus-implement --backend     # models + services + controllers
/sus-implement --frontend    # views + helpers + frontend (CSS/JS)
/sus-implement --routes      # 라우트만
/sus-implement --model       # 모델만
/sus-implement --service     # 서비스만
```

### 테스트 명령어

```bash
/sus-test system             # System 테스트 템플릿
/sus-test unit User          # 모델 Unit 테스트
/sus-test service OrderCreator  # 서비스 테스트
/sus-test js quiz_controller # Stimulus용 Vitest
/sus-test scenario           # 실행 + 오류 자동 수정
```

## 디렉토리 구조

```
.claude/commands/sus/
├── sus.md                    # 메인 라우터
├── sus-setup.md              # 프로젝트 설정
├── sus-implement.md          # 구현 오케스트레이터
├── sus-test.md               # 테스팅 에이전트
├── sus-ops.md                # 운영 설정
├── sustainable-check.md      # 원칙 검사기
├── impl/                     # 구현 원칙
│   ├── routes.md             # Canonical routes, :only 필수
│   ├── models.md             # DB 접근만, constraints
│   ├── services.md           # Stateless, Result 객체
│   ├── controllers.md        # Thin, 파라미터 변환
│   ├── views.md              # @변수 1개, locals, 시맨틱 HTML
│   ├── helpers.md            # 마크업만, 안전한 생성
│   ├── frontend.md           # Stimulus, CSS 토큰
│   ├── jobs.md               # ID만 전달, 멱등성, 서비스 위임
│   ├── mailers.md            # 포맷만, deliver_later
│   └── rake_tasks.md         # 파일당 1태스크, 서비스 위임
└── sub_made/
    └── refactor-to-services.md  # Fat model → 서비스 추출
```

## 핵심 원칙

### Routes (라우트)
```ruby
# resources에 :only 필수
resources :widgets, only: [:index, :show, :create]

# 커스텀 액션 금지 → 새 리소스로 분리
resources :widget_archives, only: [:create]  # post :archive 대신
```

### Models (모델)
```ruby
class Widget < ApplicationRecord
  # 관계, 검증, 스코프만!
  has_many :parts
  validates :name, presence: true
  scope :active, -> { where(active: true) }

  # 비즈니스 로직 금지! → 서비스로
end
```

### Services (서비스)
```ruby
class Order::Creator < ApplicationService
  def initialize(user:, items:)
    @user = user
    @items = items
  end

  def call
    order = Order.create!(user: @user, items: @items)
    success(order: order)
  rescue => e
    failure(e.message)
  end
end
```

### Controllers (컨트롤러)
```ruby
class OrdersController < ApplicationController
  def create
    result = Order::Creator.call(user: current_user, items: params[:items])

    if result.success?
      redirect_to order_path(result.value[:order])
    else
      render :new, alert: result.error
    end
  end
end
```

### Views (뷰)
```erb
<%# 파셜에는 반드시 locals 사용 %>
<%= render partial: "widget_card", locals: { widget: @widget } %>

<%# 시맨틱 HTML %>
<article>
  <header><h1><%= @widget.name %></h1></header>
  <section><%= @widget.description %></section>
</article>
```

### Testing (테스트)
```ruby
# data-testid로 안정적인 셀렉터
find("[data-testid='submit-button']").click

# System 테스트 = 핵심 유저 플로우만
# Unit 테스트 = 엣지 케이스, 계산 로직
# Service 테스트 = 비즈니스 로직
```

## 사전 준비

### ApplicationService 베이스 클래스

프로젝트에 다음 파일을 추가하세요:

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
    def success? = success
    def failure? = !success
  end
end
```

## 커스터마이징

각 `.md` 파일을 프로젝트에 맞게 수정할 수 있습니다:

1. **디자인 시스템** - `impl/frontend.md`에 CSS 클래스 추가
2. **테스트 셀렉터** - `sus-test.md`의 data-testid 패턴 수정
3. **서비스 네이밍** - `impl/services.md`의 명명 규칙 조정
4. **프로젝트 구조** - `sus-setup.md`의 경로 수정

## 참고 자료

- [Sustainable Web Development with Rails](https://sustainable-rails.com/) - 이 스킬의 기반이 된 책
- [Rails Guides](https://guides.rubyonrails.org/) - 공식 Rails 문서
- [Stimulus Handbook](https://stimulus.hotwired.dev/handbook/introduction) - Hotwire Stimulus 가이드

## 기여하기

1. 이 저장소를 Fork
2. 기능 브랜치 생성 (`git checkout -b feature/amazing-skill`)
3. 변경사항 커밋 (`git commit -m 'Add amazing skill'`)
4. 브랜치에 Push (`git push origin feature/amazing-skill`)
5. Pull Request 열기

## 라이선스

MIT License - 자유롭게 사용하세요!

---

**Built with Claude Code** - 지속 가능한 Rails 개발을 모두에게.
