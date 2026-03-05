# impl/mailers - Mailers 원칙

Sustainable Rails Chapter 20.1 기반 Mailers 설계 원칙.

---

## 핵심 원칙: Mailer는 이메일 포맷팅만!

Mailer는 **이메일의 형식과 내용을 구성하는 것**만 담당합니다.
비즈니스 로직은 반드시 Service에서 처리하세요.

---

## Mailer Pattern (얇은 메일러)

```ruby
# app/mailers/finance_mailer.rb
# ✅ Mailer는 이메일 구성만 담당
class FinanceMailer < ApplicationMailer
  def high_priced_widget(widget)
    @widget = widget
    @manufacturer = widget.manufacturer

    mail(
      to: "finance@example.com",
      subject: "High Priced Widget Alert: #{widget.name}"
    )
  end
end

# app/services/notification/high_price_alert.rb
# ✅ 비즈니스 로직은 서비스에서
class Notification::HighPriceAlert < ApplicationService
  def initialize(widget:)
    @widget = widget
  end

  def call
    return success(skipped: true) unless @widget.price_cents > 750_000

    FinanceMailer.high_priced_widget(@widget).deliver_later
    success(notified: true)
  end
end
```

---

## Core Rules (핵심 규칙)

### 1. Mailer에 비즈니스 로직 금지

```ruby
# ❌ BAD: Mailer에 조건문/비즈니스 로직
class FinanceMailer < ApplicationMailer
  def high_priced_widget(widget)
    return if widget.price_cents <= 750_000  # 비즈니스 로직!

    @widget = widget
    mail(to: "finance@example.com")
  end
end

# ✅ GOOD: Service에서 조건 처리 후 Mailer 호출
class Notification::HighPriceAlert < ApplicationService
  def call
    return success(skipped: true) unless @widget.price_cents > 750_000
    FinanceMailer.high_priced_widget(@widget).deliver_later
    success(notified: true)
  end
end
```

### 2. deliver_later 사용 (Mailers are Jobs)

```ruby
# ❌ BAD: 동기 발송 - 사용자 대기 시간 증가
FinanceMailer.high_priced_widget(widget).deliver_now

# ✅ GOOD: 비동기 발송 - Background Job으로 처리
FinanceMailer.high_priced_widget(widget).deliver_later
```

### 3. ActiveRecord 객체만 전달

```ruby
# ❌ BAD: Date 같은 객체 전달 - 직렬화 문제
FinanceMailer.monthly_report(user, Date.current).deliver_later

# ✅ GOOD: ActiveRecord 객체 또는 ID 전달
FinanceMailer.monthly_report(user).deliver_later

# Mailer 내부에서 날짜 계산
def monthly_report(user)
  @user = user
  @report_date = Date.current  # Mailer 내부에서 계산
  mail(to: user.email)
end
```

### 4. 인자로 전달된 값 사용 (멱등성)

```ruby
# ❌ BAD: 재시도 시 데이터가 변경되어 있을 수 있음
def price_alert(widget_id)
  @widget = Widget.find(widget_id)
  @price = @widget.price_cents  # 현재 가격 조회
end

# ✅ GOOD: 잡 생성 시점의 가격을 인자로 전달
# Service에서 호출 시:
FinanceMailer.price_alert(widget, widget.price_cents).deliver_later

def price_alert(widget, price_at_alert)
  @widget = widget
  @price = price_at_alert  # 고정된 값 사용
  mail(to: "finance@example.com")
end
```

---

## Mailer Preview (브라우저에서 확인)

```ruby
# test/mailers/previews/finance_mailer_preview.rb
class FinanceMailerPreview < ActionMailer::Preview
  def high_priced_widget
    # FactoryBot으로 테스트 데이터 생성 (저장 안 함)
    manufacturer = FactoryBot.build(:manufacturer, name: "Acme Corp")
    widget = FactoryBot.build(:widget,
      id: 1234,
      name: "Premium Widget",
      price_cents: 8100_00,
      manufacturer: manufacturer
    )
    FinanceMailer.high_priced_widget(widget)
  end
end

# 브라우저에서 확인:
# http://localhost:3000/rails/mailers/finance_mailer/high_priced_widget
```

---

## Development 환경 설정

### MailCatcher 사용

```ruby
# config/environments/development.rb
config.action_mailer.delivery_method = :smtp
config.action_mailer.smtp_settings = { address: "localhost", port: 1025 }
```

```bash
# MailCatcher 설치 및 실행
gem install mailcatcher
mailcatcher

# 웹 인터페이스: http://localhost:1080
```

### Resend 사용 (Production)

```ruby
# config/environments/production.rb
config.action_mailer.delivery_method = :smtp
config.action_mailer.smtp_settings = {
  address: "smtp.resend.com",
  port: 587,
  user_name: "resend",
  password: ENV["RESEND_API_KEY"],
  authentication: :plain,
  enable_starttls_auto: true
}
```

---

## Checklist

- [ ] **비즈니스 로직 분리** (Service에서 조건 처리)
- [ ] **deliver_later 사용** (비동기 발송)
- [ ] **ActiveRecord 객체만 전달** (직렬화 안전)
- [ ] **값 고정** (재시도 시 데이터 변경 대비)
- [ ] **Preview 작성** (브라우저에서 확인)
- [ ] **I18n 사용** (제목, 본문 번역)
