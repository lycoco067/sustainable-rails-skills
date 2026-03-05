# /sus-ops - Sustainable Rails Operations Agent

"Sustainable Web Development with Rails" Part 3 기반의 운영 에이전트.
Authentication, API Design, Logging, Exception Handling, Monitoring을 다룹니다.

## Trigger Keywords
- sus ops, operations
- auth, authentication, devise
- api design, api versioning
- logging, sentry, monitoring

---

## 1. AUTHENTICATION

### 1.1 Devise Setup
```ruby
# app/models/user.rb
class User < ApplicationRecord
  devise :database_authenticatable, :registerable,
         :recoverable, :rememberable, :validatable, :trackable,
         :omniauthable, omniauth_providers: [:google_oauth2]
end
```

### 1.2 Auth Checklist
- [ ] Devise 설치 및 설정
- [ ] OAuth providers 설정 (Google, etc.)
- [ ] 세션 타임아웃 설정
- [ ] Remember me 기능

### 1.3 Authorization (권한 부여)

인증(Authentication)이 "누구인가?"라면, **권한 부여(Authorization)**는 "무엇을 할 수 있는가?"입니다.

#### 권장 선택 가이드

| 상황 | 권장 |
|------|------|
| 단순 admin/user | Rails 네이티브 |
| 중간 복잡도 | Pundit |
| 복잡한 역할 | Pundit 또는 Action Policy |
| 다국어 지원 필요 | Action Policy |

#### Option 1: Rails 네이티브 (단순한 경우)
```ruby
class AdminController < ApplicationController
  before_action :require_admin

  private

  def require_admin
    unless current_user&.admin?
      redirect_to root_path, alert: "권한이 없습니다"
    end
  end
end
```

#### Option 2: Pundit (권장)
```ruby
# Gemfile
gem "pundit"

# app/policies/resource_policy.rb
class ResourcePolicy < ApplicationPolicy
  def show?
    record.user_id == user.id
  end

  def update?
    record.user_id == user.id
  end
end

# Controller에서 사용
def show
  @resource = Resource.find(params[:id])
  authorize @resource
end
```

---

## 2. API DESIGN

### 2.1 Resource-Oriented Routes
```ruby
# config/routes.rb
namespace :api do
  namespace :v1 do
    resources :widgets, only: [:index, :show, :create, :update]
    resource :profile, only: [:show, :update]
  end
end
```

### 2.2 API Base Controller
```ruby
module Api
  class BaseController < ActionController::API
    before_action :validate_content_type

    rescue_from ActiveRecord::RecordNotFound, with: :not_found
    rescue_from ActionController::ParameterMissing, with: :bad_request

    private

    def validate_content_type
      return if request.get? || request.delete?

      unless request.content_type&.include?("application/json")
        render json: { error: "Content-Type must be application/json" },
               status: :unsupported_media_type
      end
    end

    def not_found
      render json: { error: "Resource not found" }, status: :not_found
    end

    def bad_request(exception)
      render json: { error: exception.message }, status: :bad_request
    end
  end
end
```

### 2.3 JSON Response Pattern
```ruby
def show
  @widget = Widget.find(params[:id])
  render json: @widget.as_json(
    only: [:id, :name, :description],
    include: { category: { only: [:id, :name] } }
  )
end

# With metadata
def index
  @widgets = Widget.page(params[:page])
  render json: {
    data: @widgets.as_json(only: [:id, :name]),
    meta: {
      total: @widgets.total_count,
      page: @widgets.current_page,
      per_page: @widgets.limit_value
    }
  }
end
```

### 2.4 API Versioning
```
URL versioning: /api/v1/widgets  ✅ (recommended)
Query param:    /api/widgets?v=1 ❌
Header:         Accept: api/v1   ❌

Breaking changes → New version (v2)
Non-breaking → Same version
```

---

## 3. OBSERVABILITY (관측 가능성)

### 3.1 Three Pillars of Observability

| Pillar | 설명 | Rails 구현 |
|--------|------|-----------|
| **Logs** | 이벤트 기록 | Lograge + Request ID |
| **Metrics** | 숫자 측정값 | ActiveSupport::Notifications |
| **Traces** | 요청 흐름 추적 | Request ID 연결 |

### 3.2 비즈니스 결과 중심 모니터링

```ruby
# ❌ 기술 메트릭만
"HTTP 500 errors: 5"
"Response time: 150ms"

# ✅ 비즈니스 결과 중심
"Daily active users: 1,234"
"Feature completion rate: 89%"
"Conversion rate: 78%"
```

---

## 4. LOGGING

### 4.1 로깅 4대 핵심 기법

| 기법 | 설명 | 예시 |
|------|------|------|
| **1. Request ID** | 요청 전체 추적 | `request_id=abc123` |
| **2. Identifiers** | 관련 객체 식별 | `user_id=42`, `order_id=789` |
| **3. Source** | 로그 발생 위치 | `[OrderService]` |
| **4. User ID** | 사용자 식별 | `user_id=42` |

### 4.2 Lograge Setup
```ruby
# config/initializers/lograge.rb
Rails.application.configure do
  config.lograge.enabled = true
  config.lograge.formatter = Lograge::Formatters::Json.new

  config.lograge.custom_options = lambda do |event|
    {
      request_id: event.payload[:request_id],
      user_id: event.payload[:user_id],
      host: event.payload[:host],
      ip: event.payload[:ip]
    }.compact
  end

  config.lograge.custom_payload do |controller|
    {
      host: controller.request.host,
      ip: controller.request.remote_ip,
      user_id: controller.try(:current_user)&.id
    }
  end
end
```

### 4.3 Application Controller Integration
```ruby
class ApplicationController < ActionController::Base
  before_action :set_request_context

  private

  def set_request_context
    Current.request_id = request.request_id
    Current.user = current_user
  end

  def append_info_to_payload(payload)
    super
    payload[:request_id] = request.request_id
    payload[:user_id] = current_user&.id
  end
end
```

---

## 5. EXCEPTION HANDLING

### 5.1 Sentry Setup
```ruby
# Gemfile
gem "sentry-ruby"
gem "sentry-rails"

# config/initializers/sentry.rb
Sentry.init do |config|
  config.dsn = ENV["SENTRY_DSN"]
  config.breadcrumbs_logger = [:active_support_logger, :http_logger]
  config.traces_sample_rate = Rails.env.production? ? 0.1 : 1.0
  config.environment = Rails.env
  config.send_default_pii = false
  config.release = ENV.fetch("GIT_COMMIT", "unknown")

  config.excluded_exceptions += [
    "ActionController::RoutingError",
    "ActiveRecord::RecordNotFound"
  ]
end
```

### 5.2 Context Enhancement
```ruby
class ApplicationController < ActionController::Base
  before_action :set_sentry_context

  private

  def set_sentry_context
    Sentry.set_user(
      id: current_user&.id,
      email: current_user&.email
    )

    Sentry.set_context("request", {
      request_id: request.request_id,
      locale: I18n.locale.to_s,
      user_agent: request.user_agent
    })
  end
end
```

---

## 6. MONITORING

### 6.1 Health Check Endpoint
```ruby
# config/routes.rb
get "up" => "rails/health#show"
get "health" => "health#show"

# app/controllers/health_controller.rb
class HealthController < ApplicationController
  skip_before_action :authenticate_user!

  def show
    checks = {
      database: database_connected?,
      redis: redis_connected?
    }

    status = checks.values.all? ? :ok : :service_unavailable
    render json: { status: status, checks: checks }, status: status
  end

  private

  def database_connected?
    ActiveRecord::Base.connection.active?
  rescue
    false
  end

  def redis_connected?
    Redis.current.ping == "PONG"
  rescue
    false
  end
end
```

---

## 7. SECRETS MANAGEMENT

### 7.1 Rails Credentials
```bash
bin/rails credentials:edit
bin/rails credentials:edit --environment production
```

```yaml
# config/credentials.yml.enc
sentry:
  dsn: https://xxx@sentry.io/123

google_oauth:
  client_id: xxx.apps.googleusercontent.com
  client_secret: GOCSPX-xxx
```

### 7.2 Usage
```ruby
# Access credentials
Rails.application.credentials.dig(:sentry, :dsn)
Rails.application.credentials.dig(:google_oauth, :client_id)
```

---

## Operations Commands

| Command | Action |
|---------|--------|
| `/sus-ops` | 운영 개요 |
| `/sus-ops auth` | 인증 설정 체크리스트 |
| `/sus-ops api` | API 설계 가이드 |
| `/sus-ops logging` | 로깅 4대 기법 |
| `/sus-ops sentry` | Sentry 설정 |
| `/sus-ops monitor` | 모니터링 가이드 |
| `/sus-ops secrets` | Rails Credentials 가이드 |
| `/sus-ops ci` | CI 설정 가이드 |
