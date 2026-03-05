# /sus-test - Sustainable Rails Testing Agent

"Sustainable Web Development with Rails" 테스팅 전략 기반의 테스트 에이전트.
System tests, Model tests, Service tests, JavaScript tests를 다룹니다.

## Trigger Keywords
- sus test, write tests
- system tests, unit tests
- coverage, data-testid
- vitest, test strategy

---

## Testing Pyramid

```
                ┌─────────────┐
               /   System     \        Few, high-value
              /    Tests       \       Major user flows only
             /   (Capybara)     \
            /___________________\
           /                     \
          /    Integration        \    Moderate
         /       Tests             \   API, Controller
        /_________________________\
       /                           \
      /         Unit Tests          \  Many, fast
     /     (Models, Services)        \
    /________________________________\
   /                                   \
  /       JavaScript Unit Tests         \  Critical paths
 /           (Vitest + jsdom)            \
/________________________________________\
```

---

## 1. SYSTEM TESTS

### 1.1 When to Write
- **Major user flows only** - 로그인, 핵심 기능 완료, 결제 등
- **Not for edge cases** - unit tests에서 처리
- **High business value** - 핵심 기능

### 1.2 Configuration
```ruby
# test/application_system_test_case.rb
class ApplicationSystemTestCase < ActionDispatch::SystemTestCase
  # JavaScript 필요 시 headless Chrome
  driven_by :selenium, using: :headless_chrome, screen_size: [1400, 900]

  # JavaScript 불필요 시 rack_test (더 빠름)
  # driven_by :rack_test
end
```

### 1.3 data-testid Pattern
```erb
<!-- View: data-testid 속성 사용 -->
<button data-testid="submit-button" class="btn btn-primary">
  Submit
</button>

<div data-testid="success-message" class="alert">
  Success!
</div>

<span data-testid="item-count"><%= @items.count %></span>
```

```ruby
# Test: data-testid로 요소 찾기
test "user completes action successfully" do
  visit some_path

  # data-testid 사용 (CSS 변경에 안전)
  find("[data-testid='item-1']").click
  find("[data-testid='option-A']").click
  find("[data-testid='submit-button']").click

  assert_selector "[data-testid='success-message']"
end
```

### 1.4 System Test Template
```ruby
# test/system/feature_flow_test.rb
require "application_system_test_case"

class FeatureFlowTest < ApplicationSystemTestCase
  setup do
    @resource = resources(:example)
  end

  test "user can complete the flow" do
    visit start_path

    # Step 1
    find("[data-testid='step-1-button']").click

    # Step 2
    fill_in "Name", with: "Test Name"
    find("[data-testid='continue-button']").click

    # Verify
    assert_selector "[data-testid='success-message']"
    assert_text "Completed"
  end

  test "user sees error on invalid input" do
    visit start_path

    # Submit without required field
    find("[data-testid='submit-button']").click

    # Verify error
    assert_selector "[data-testid='error-message']"
  end
end
```

### 1.5 Fake Backend First
```ruby
# 외부 서비스 모킹
require "webmock/minitest"

class ExternalApiTest < ApplicationSystemTestCase
  setup do
    stub_request(:post, "https://api.example.com/endpoint")
      .to_return(status: 200, body: '{"id": "123"}')
  end

  test "sends request to external API" do
    # ...
  end
end
```

---

## 2. SERVICE TESTS

### 2.1 Service Test Pattern
```ruby
# test/services/namespace/service_name_test.rb
require "test_helper"

module Namespace
  class ServiceNameTest < ActiveSupport::TestCase
    setup do
      @user = users(:default_user)
    end

    test "succeeds with valid input" do
      result = ServiceName.call(
        user: @user,
        param: "value"
      )

      assert result.success?
      assert_not_nil result.value[:output]
    end

    test "fails with invalid input" do
      result = ServiceName.call(
        user: nil,
        param: "value"
      )

      assert result.failure?
      assert_includes result.error.downcase, "user"
    end

    test "handles edge case gracefully" do
      result = ServiceName.call(
        user: @user,
        param: ""
      )

      assert result.success?
      # Behavior depends on business rules
    end
  end
end
```

### 2.2 Test Boundaries
```ruby
# ✅ Test inputs/outputs, not implementation
test "calculates result correctly" do
  result = Calculator.call(
    user: @user,
    input: 100
  )

  assert result.success?
  assert_equal 75, result.value[:output]
end

# ❌ Don't test internal implementation details
# test "calls the database exactly 2 times" - BAD
```

---

## 3. MODEL TESTS

### 3.1 Validation Tests
```ruby
# test/models/user_test.rb
require "test_helper"

class UserTest < ActiveSupport::TestCase
  test "requires email" do
    user = User.new(email: nil)
    assert_not user.valid?
    assert_includes user.errors[:email], "can't be blank"
  end

  test "requires unique email" do
    existing = users(:default_user)
    user = User.new(email: existing.email)
    assert_not user.valid?
    assert_includes user.errors[:email], "has already been taken"
  end

  test "validates format" do
    user = User.new(field: "invalid@format!")
    assert_not user.valid?

    user.field = "valid_format"
    user.email = "test@example.com"
    user.password = "password123"
    assert user.valid?
  end
end
```

### 3.2 Scope Tests
```ruby
test "active scope returns only active records" do
  active = records(:active)
  inactive = records(:inactive)

  result = Record.active

  assert_includes result, active
  assert_not_includes result, inactive
end
```

### 3.3 Database Constraint Tests
```ruby
test "database enforces unique constraint" do
  # First record
  Record.create!(
    field_a: "value",
    field_b: "other"
  )

  # Second record with same keys should fail
  error = assert_raises(ActiveRecord::RecordNotUnique) do
    Record.create!(
      field_a: "value",
      field_b: "other"
    )
  end

  assert_match /unique/i, error.message
end
```

---

## 4. CONTROLLER TESTS

### 4.1 Integration Tests
```ruby
# test/controllers/api/v1/resource_controller_test.rb
require "test_helper"

class Api::V1::ResourceControllerTest < ActionDispatch::IntegrationTest
  test "returns resource" do
    get api_v1_resource_path, params: { id: 1 }

    assert_response :success
    json = JSON.parse(response.body)
    assert json.key?("data")
  end

  test "returns 400 without required param" do
    get api_v1_resource_path

    assert_response :bad_request
  end

  test "creates resource via API" do
    post api_v1_resources_path, params: {
      name: "Test",
      value: 100
    }, as: :json

    assert_response :success
    json = JSON.parse(response.body)
    assert json["created"]
  end
end
```

---

## 5. JAVASCRIPT TESTS (Vitest)

### 5.1 Setup
```javascript
// vitest.config.js
import { defineConfig } from 'vitest/config'

export default defineConfig({
  test: {
    environment: 'jsdom',
    include: ['app/javascript/**/*.test.js'],
    globals: true,
  },
})
```

```json
// package.json
{
  "scripts": {
    "test": "vitest",
    "test:run": "vitest run"
  },
  "devDependencies": {
    "vitest": "^1.0.0",
    "jsdom": "^23.0.0"
  }
}
```

### 5.2 Stimulus Controller Test
```javascript
// app/javascript/controllers/example_controller.test.js
import { describe, it, expect, beforeEach } from 'vitest'
import { Application } from '@hotwired/stimulus'
import ExampleController from './example_controller'

describe('ExampleController', () => {
  let application

  beforeEach(() => {
    document.body.innerHTML = `
      <div data-controller="example" data-example-count-value="5">
        <span data-example-target="display">5</span>
      </div>
    `
    application = Application.start()
    application.register('example', ExampleController)
  })

  it('displays initial value', () => {
    const display = document.querySelector('[data-example-target="display"]')
    expect(display.textContent).toBe('5')
  })

  it('updates value on action', async () => {
    const controller = application.getControllerForElementAndIdentifier(
      document.querySelector('[data-controller="example"]'),
      'example'
    )

    controller.increment()

    expect(controller.countValue).toBe(6)
  })
})
```

### 5.3 Utility Function Tests
```javascript
// app/javascript/utils/calculator.test.js
import { describe, it, expect } from 'vitest'
import { calculate, format } from './calculator'

describe('calculate', () => {
  it('returns correct result', () => {
    expect(calculate(10, 10)).toBe(100)
  })

  it('handles zero', () => {
    expect(calculate(0, 10)).toBe(0)
  })

  it('rounds correctly', () => {
    expect(calculate(1, 3)).toBe(33)
  })
})

describe('format', () => {
  it('formats with suffix', () => {
    expect(format(85)).toBe('85%')
  })
})
```

---

## 6. FIXTURES vs FACTORIES

### 6.1 Fixtures (Rails Default)
```yaml
# test/fixtures/users.yml
default_user:
  email: test@example.com
  name: tester
  encrypted_password: <%= Devise::Encryptor.digest(User, 'password123') %>

admin_user:
  email: admin@example.com
  name: admin
  admin: true
```

### 6.2 FactoryBot (Complex Scenarios)
```ruby
# test/factories/users.rb
FactoryBot.define do
  factory :user do
    sequence(:email) { |n| "user#{n}@example.com" }
    password { "password123" }
    name { Faker::Internet.username(specifier: 5..10) }

    trait :admin do
      admin { true }
    end

    trait :with_data do
      after(:create) do |user|
        create(:user_data, user: user)
      end
    end
  end
end
```

---

## Test Commands

| Command | Action |
|---------|--------|
| `/sus-test` | 테스트 전략 개요 |
| `/sus-test system` | System test 템플릿 생성 |
| `/sus-test service <name>` | Service test 생성 |
| `/sus-test model <name>` | Model test 생성 |
| `/sus-test controller <name>` | Controller test 생성 |
| `/sus-test js` | Vitest 설정 및 JS test 생성 |
| `/sus-test coverage` | 커버리지 리포트 실행 |
| `/sus-test scenario` | **시나리오 병렬 실행 + 오류 자동 수정 + 캡쳐** |
| `/sus-test journey` | **유저 저니 스크린샷 캡쳐** |

---

## 7. USER JOURNEY SCREENSHOT

### 7.1 개요

유저 저니를 따라가며 스크린샷을 캡쳐합니다.

### 7.2 스크린샷 저장 경로

```
screenshots/{journey_name}/
├── 01_step_name.png
├── 02_step_name.png
└── ...
```

### 7.3 테스트 템플릿

```ruby
# test/system/{feature}_journey_test.rb
class FeatureJourneyTest < ApplicationSystemTestCase
  SCREENSHOT_DIR = Rails.root.join("screenshots", "feature_journey")

  setup do
    FileUtils.mkdir_p(SCREENSHOT_DIR)
  end

  teardown do
    if passed?
      puts "\n📸 스크린샷 저장 완료: #{SCREENSHOT_DIR}"
      system("open", SCREENSHOT_DIR.to_s)  # Finder에서 자동 열기
    end
  end

  test "capture user journey" do
    visit some_path
    page.save_screenshot(SCREENSHOT_DIR.join("01_step.png"))
    # ...
  end
end
```

---

## 8. SCENARIO PARALLEL EXECUTION

### 8.1 개요
시나리오 케이스를 입력받아 각 시나리오를 **독립적인 Task로 병렬 실행**합니다.
오류 발견 시 **자동 수정** 후 재실행하고, 성공 시 **스크린샷을 캡쳐**합니다.

```
시나리오 입력 → [Task 1] [Task 2] [Task 3] ... (병렬)
                   ↓        ↓        ↓
              테스트 → 오류? → 자동수정 → 재실행 → 성공 시 캡쳐
                   ↓
              결과 취합 → 보고서 생성
```

### 8.2 사용법

```bash
# YAML 파일 기반 실행
/sus-test scenario @test/scenarios/scenarios.yml

# 인라인 시나리오 실행
/sus-test scenario "
1. 랜딩→상세 이동
2. 기능 완료
3. 로그인 성공
"

# 특정 시나리오만 실행
/sus-test scenario --only "기능 완료"
```

### 8.3 시나리오 입력 형식 (YAML)
```yaml
scenarios:
  - name: "기본 플로우"
    file: basic_flow_test.rb
    steps:
      - visit root_path
      - click "시작하기"
      - expect result_path
```

### 8.4 Auto-Fix 워크플로우
각 Task 내부에서 최대 3회까지 자동 수정 시도:
1. 테스트 실행
2. 오류 발생 시 분석 → 코드 수정
3. 재실행
4. 성공 시 스크린샷 캡쳐
5. **Finder에서 스크린샷 폴더 자동 열기**

### 8.5 완료 후 자동 열기
```ruby
# 테스트 완료 후 Finder에서 스크린샷 폴더 열기
system("open", SCREENSHOT_DIR.to_s) if passed?
```

```bash
# 또는 Bash에서
open screenshots/{journey_name}/
```

### 8.6 오류 자동 수정 패턴
| 오류 유형 | 자동 수정 방법 |
|-----------|----------------|
| `ElementNotFound` | data-testid 추가 |
| `AssertionError` | 예상값 확인 후 수정 |
| `Timeout` | wait 시간 증가 |
| `RoutingError` | routes.rb 추가 |

---

## TDD Workflow

```
1. Red   → 실패하는 테스트 작성
2. Green → 테스트 통과하는 최소 코드
3. Refactor → 코드 정리 (테스트 계속 통과)
```

### Example TDD Flow
```ruby
# Step 1: Red - Write failing test
test "calculates value correctly" do
  result = Calculator.call(input: 100)
  assert_equal 75, result.value[:output]
end
# => NameError: uninitialized constant Calculator

# Step 2: Green - Minimal implementation
class Calculator < ApplicationService
  def initialize(input:)
    @input = input
  end

  def call
    success(output: 75)  # Hardcoded to pass
  end
end
# => Test passes

# Step 3: Refactor - Real implementation
def call
  calculated = @input * 0.75
  success(output: calculated.to_i)
end
```

---

## Running Tests

```bash
# All tests
bin/rails test

# System tests only
bin/rails test:system

# Specific file
bin/rails test test/models/user_test.rb

# Specific test (line number)
bin/rails test test/models/user_test.rb:15

# JavaScript tests
yarn test        # Watch mode
yarn test:run    # Single run

# Full CI suite
bin/ci
```

---

## data-testid Best Practices

1. **Add only when needed** (Rule of Three)
2. **Use descriptive names**: `submit-form` not `btn1`
3. **Prefix by context**: `user-card`, `form-input`
4. **Past tense for states**: `item-completed`, `form-submitted`
5. **Don't use for styling** - data-testid is for tests only
