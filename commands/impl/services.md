# impl/services - Business Logic 원칙

Sustainable Rails Service Layer 설계 원칙.

---

## Service Pattern (ApplicationService)

```ruby
# app/services/order/creator.rb
module Order
  class Creator < ApplicationService
    def initialize(user:, items:, **options)
      @user = user
      @items = items
      @coupon_code = options[:coupon_code]
    end

    def call
      order = Order.create!(
        user: @user,
        items: @items,
        total_cents: calculate_total
      )

      success(order: order)
    rescue ActiveRecord::RecordInvalid => e
      failure(e.message)
    end

    private

    def calculate_total
      # 비즈니스 로직
    end
  end
end
```

---

## Service Rules

```ruby
# 1. Stateless - 초기화 후 인스턴스 변수 변경 없음
# 2. Explicit params - 명시적 파라미터
# 3. Rich Result - boolean 아닌 Result 객체 반환
# 4. Single responsibility - 하나의 작업만
```

---

## Controller에서 사용

```ruby
def create
  result = Order::Creator.call(
    user: current_user,
    items: params[:items],
    coupon_code: params[:coupon_code]
  )

  if result.success?
    redirect_to order_path(result.value[:order])
  else
    render :new, alert: result.error
  end
end
```

---

## Service Naming Convention

| 동작 | 네이밍 패턴 | 예시 |
|------|------------|------|
| 생성/처리 | `*Creator`, `*Handler` | `OrderCreator` |
| 계산 | `*Calculator` | `ProgressCalculator` |
| 조회 | `*Finder`, `*Query` | `LeaderboardFinder` |
| 상태변경 | `*Updater`, `*Manager` | `InventoryManager` |
| 검증 | `*Validator` | `PaymentValidator` |
| 알림 | `*Notifier`, `*Sender` | `OrderNotifier` |

---

## Complex Logic → /refactor-to-services

Fat model 발견 시: `/sus refactor` 또는 `/refactor-to-services` 사용

---

## Checklist

- [ ] Stateless 클래스
- [ ] 명시적 named parameters
- [ ] Rich Result 객체 반환 (success/failure)
- [ ] Single responsibility
- [ ] 네이밍 컨벤션 준수
