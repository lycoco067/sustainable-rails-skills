# impl/controllers - Controllers 원칙

Sustainable Rails Controllers 설계 원칙.

---

## Thin Controllers

```ruby
class WidgetsController < ApplicationController
  def show
    @widget = Widget.find(params[:id])
    # That's it! 비즈니스 로직 없음
  end

  def create
    result = Widget::Creator.call(
      user: current_user,
      params: widget_params
    )

    if result.success?
      redirect_to widget_path(result.value[:widget])
    else
      render :new, alert: result.error
    end
  end
end
```

---

## Parameter Conversion

```ruby
# 문자열 → 적절한 타입
def create
  widget_params = params.require(:widget).permit(:name, :price)

  # 달러 → 센트 변환
  if widget_params[:price].present?
    widget_params[:price_cents] = (BigDecimal(widget_params[:price]) * 100).to_i
    widget_params.delete(:price)
  end

  # Service 호출
  result = Widget::Creator.call(**widget_params.to_h.symbolize_keys)
  # ...
end
```

---

## Minimal Callbacks

```ruby
class ApplicationController < ActionController::Base
  # ✅ OK: 인증, 로깅
  before_action :authenticate_user!, except: [:index, :show]
  before_action :set_locale

  # ❌ NO: 비즈니스 로직
  # before_action :check_subscription  # BAD - 서비스 또는 concern으로
end
```

---

## Controller Responsibilities

```
1. HTTP 요청 수신 (params = "hashes of strings")
2. 파라미터 → 리치 타입 변환
3. 서비스 호출
4. 결과에 따라 redirect/render
```

---

## Checklist

- [ ] 컨트롤러는 thin (비즈니스 로직 없음)
- [ ] 서비스 클래스로 위임
- [ ] 파라미터 변환은 컨트롤러에서
- [ ] 콜백은 인증/로깅만
