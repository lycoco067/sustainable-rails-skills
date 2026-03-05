# /refactor-to-services - 서비스 레이어 리팩토링 에이전트

Active Record 모델에서 비즈니스 로직을 분리하여 Service Layer 패턴으로 리팩토링합니다.
"Sustainable Web Development with Rails" 책의 Chapter 2 원칙을 따릅니다.

## Trigger Keywords
- 서비스 레이어, service layer
- 비즈니스 로직 분리, 리팩토링
- fat model, 모델 정리

---

## 핵심 원칙

### Fan-in 최소화
```
높은 Fan-in 클래스 (User, Widget 등)
  → 버그 영향 범위 넓음
  → 비즈니스 로직 넣지 말 것!

낮은 Fan-in 클래스 (서비스 클래스들)
  → 버그 영향 범위 좁음
  → 비즈니스 로직은 여기에!
```

### Active Record는 DB 접근만
```ruby
# ✅ Good: Active Record
class User < ApplicationRecord
  has_many :orders
  belongs_to :organization

  validates :email, presence: true

  scope :active, -> { where(active: true) }
end

# ❌ Bad: 비즈니스 로직이 모델에 있음
class User < ApplicationRecord
  def calculate_weekly_progress
    # 복잡한 로직...
  end

  def send_reminder
    # 외부 서비스 호출...
  end
end
```

---

## Phase 1: 프로젝트 분석

### 1.1 모델 파일 스캔
```bash
# 모든 모델 파일 확인
ls -la app/models/

# 각 모델의 라인 수 확인 (큰 파일 = 리팩토링 대상)
wc -l app/models/*.rb | sort -n
```

### 1.2 비즈니스 로직 탐지 패턴

다음 패턴이 모델에 있으면 서비스로 분리 대상:

| 패턴 | 설명 | 예시 |
|------|------|------|
| 외부 API 호출 | HTTP 요청, 메일 발송 | `HTTParty`, `Mailer` |
| 복잡한 계산 | 여러 모델 조합 로직 | `calculate_*`, `compute_*` |
| 상태 변경 + 부수효과 | 저장 + 알림 등 | `process_*`, `handle_*` |
| 조건부 비즈니스 규칙 | if/else 체인 | `determine_*`, `decide_*` |
| 트랜잭션 조율 | 여러 모델 동시 변경 | `transaction do` |
| 외부 서비스 연동 | 결제, 알림 등 | `notify_*`, `charge_*` |

### 1.3 분석 체크리스트

각 모델에 대해 확인:
- [ ] 50줄 이상인가?
- [ ] `def self.` 메서드가 3개 이상인가?
- [ ] 인스턴스 메서드가 5개 이상인가?
- [ ] 콜백에 복잡한 로직이 있는가?
- [ ] 다른 모델을 직접 생성/수정하는가?
- [ ] 외부 서비스를 호출하는가?

---

## Phase 2: 서비스 클래스 설계

### 2.1 폴더 구조 생성

```
app/
├── models/           # DB 접근만
│   ├── user.rb
│   ├── order.rb
│   └── product.rb
│
├── services/         # 비즈니스 로직
│   ├── order/
│   │   ├── creator.rb
│   │   ├── canceller.rb
│   │   └── refunder.rb
│   │
│   ├── user/
│   │   ├── registrar.rb
│   │   ├── authenticator.rb
│   │   └── profile_updater.rb
│   │
│   └── notification/
│       ├── email_sender.rb
│       └── push_notifier.rb
```

### 2.2 서비스 클래스 네이밍 규칙

| 동작 | 네이밍 패턴 | 예시 |
|------|------------|------|
| 생성/처리 | `*Creator`, `*Handler` | `OrderCreator` |
| 계산 | `*Calculator` | `ProgressCalculator` |
| 조회 | `*Finder`, `*Query` | `LeaderboardFinder` |
| 상태변경 | `*Updater`, `*Manager` | `InventoryManager` |
| 검증 | `*Validator` | `PaymentValidator` |
| 알림 | `*Notifier`, `*Sender` | `OrderNotifier` |

### 2.3 서비스 클래스 템플릿

```ruby
# app/services/order/creator.rb

module Order
  class Creator
    def initialize(user:, items:)
      @user = user
      @items = items
    end

    def call
      ActiveRecord::Base.transaction do
        create_order
        reserve_inventory
        send_confirmation
      end

      Result.new(success: true, order: @order)
    rescue => e
      Result.new(success: false, error: e.message)
    end

    private

    attr_reader :user, :items

    def create_order
      @order = Order.create!(
        user: user,
        items: items,
        total_cents: calculate_total
      )
    end

    def reserve_inventory
      # ...
    end

    def send_confirmation
      # ...
    end

    # 결과 객체
    Result = Struct.new(:success, :order, :error, keyword_init: true) do
      def success?
        success
      end
    end
  end
end
```

---

## Phase 3: 리팩토링 실행

### 3.1 단계별 마이그레이션

```
1. 서비스 클래스 생성 (새 파일)
2. 모델에서 로직 복사 → 서비스로
3. 컨트롤러에서 서비스 호출로 변경
4. 테스트 작성/수정
5. 모델에서 기존 메서드 제거
6. 테스트 실행 확인
```

### 3.2 컨트롤러 변경 예시

```ruby
# ❌ Before: 컨트롤러가 모델 메서드 직접 호출
class OrdersController < ApplicationController
  def create
    @user.create_order(params[:items])
    @user.update_inventory
    @user.send_confirmation
  end
end

# ✅ After: 서비스 클래스 사용
class OrdersController < ApplicationController
  def create
    result = Order::Creator.new(
      user: current_user,
      items: params[:items]
    ).call

    if result.success?
      redirect_to order_path(result.order), notice: "주문 완료!"
    else
      render :new, alert: result.error
    end
  end
end
```

### 3.3 모델 정리 예시

```ruby
# ❌ Before: Fat Model
class User < ApplicationRecord
  has_many :orders

  def create_order(items)
    # 30줄의 복잡한 로직...
  end

  def update_inventory
    # 20줄의 로직...
  end

  def calculate_weekly_stats
    # 25줄의 로직...
  end
end

# ✅ After: Thin Model
class User < ApplicationRecord
  has_many :orders

  # DB 관계, 검증, 스코프만
  validates :email, presence: true
  scope :active, -> { where(active: true) }
end
```

---

## Phase 4: 테스트 작성

### 4.1 서비스 테스트 템플릿

```ruby
# test/services/order/creator_test.rb

require "test_helper"

module Order
  class CreatorTest < ActiveSupport::TestCase
    setup do
      @user = users(:default_user)
      @items = [{ product_id: 1, quantity: 2 }]
    end

    test "성공적으로 주문을 생성한다" do
      result = Creator.new(
        user: @user,
        items: @items
      ).call

      assert result.success?
      assert_not_nil result.order
    end

    test "재고가 부족하면 에러를 반환한다" do
      @items = [{ product_id: 1, quantity: 9999 }]

      result = Creator.new(
        user: @user,
        items: @items
      ).call

      assert_not result.success?
      assert_not_nil result.error
    end

    test "잘못된 입력 시 에러를 반환한다" do
      result = Creator.new(
        user: nil,
        items: @items
      ).call

      assert_not result.success?
      assert_not_nil result.error
    end
  end
end
```

---

## 실행 워크플로우

### Step 1: 분석
```
1. app/models/ 전체 스캔
2. 각 모델의 메서드 목록 추출
3. 비즈니스 로직 후보 식별
4. 리팩토링 우선순위 결정
```

### Step 2: 계획 수립
```
1. 서비스 클래스 목록 작성
2. 각 서비스의 책임 정의
3. 의존성 파악
4. 마이그레이션 순서 결정
```

### Step 3: 실행
```
1. app/services/ 폴더 생성
2. 서비스 클래스 하나씩 생성
3. 테스트 작성
4. 컨트롤러 수정
5. 모델에서 제거
6. 전체 테스트 실행
```

### Step 4: 검증
```
1. bin/rails test 전체 통과
2. 기능 테스트 (브라우저)
3. 코드 리뷰
```

---

## 주의사항

### 하지 말아야 할 것
- ❌ 한 번에 모든 것을 리팩토링
- ❌ 테스트 없이 리팩토링
- ❌ 기존 API 변경 (점진적으로)
- ❌ 서비스에서 또 다른 서비스 과도하게 호출

### 해야 할 것
- ✅ 작은 단위로 점진적 리팩토링
- ✅ 각 단계마다 테스트 실행
- ✅ 하나의 서비스 = 하나의 책임
- ✅ 명확한 입력/출력 정의

---

## 명령어

| 명령어 | 설명 |
|--------|------|
| `analyze` | 현재 프로젝트 분석 |
| `plan` | 리팩토링 계획 수립 |
| `execute` | 리팩토링 실행 |
| `test` | 테스트 실행 |
| `report` | 결과 리포트 생성 |
