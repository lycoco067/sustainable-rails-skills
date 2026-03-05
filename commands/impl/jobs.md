# impl/jobs - Background Jobs 원칙

Sustainable Rails Chapter 19 기반 Background Jobs 설계 원칙.

---

## 왜 Background Jobs를 사용하는가?

1. **사용자 대기 시간 감소**: 느린 작업을 백그라운드로 옮겨 응답 속도 향상
2. **실패 복구 가능**: 네트워크 에러 등 실패 시 자동 재시도
3. **웹 워커 용량 확보**: Worker가 하는 일을 줄이면 처리 용량이 늘어남

---

## Job Pattern (얇은 잡)

```ruby
# app/jobs/send_reminder_job.rb
# ✅ 잡은 진입점일 뿐 - 비즈니스 로직은 서비스에!
class SendReminderJob < ApplicationJob
  queue_as :default

  def perform(user_id)
    user = User.find(user_id)
    Notification::Reminder.call(user: user)  # 서비스에 위임
  end
end

# 호출
SendReminderJob.perform_later(user.id)
```

---

## Core Rules (핵심 규칙)

### 1. ID만 전달 (객체 전달 금지)

```ruby
# ❌ BAD: 객체 전달 - 직렬화 문제 발생
ProcessOrderJob.perform_later(order)

# ✅ GOOD: ID만 전달 - 잡 안에서 조회
ProcessOrderJob.perform_later(order.id)
```

### 2. 서비스로 위임 (Fat Job 금지)

```ruby
# ❌ BAD: 잡에 비즈니스 로직
class WeeklyResetJob < ApplicationJob
  def perform
    finalize_data      # 로직 1
    process_changes    # 로직 2
    send_notifications # 로직 3
  end
end

# ✅ GOOD: 서비스에 위임
class WeeklyResetJob < ApplicationJob
  def perform
    WeeklyResetService.call
  end
end
```

### 3. 멱등성 (Idempotency) - 재시도 안전

```ruby
# ✅ 같은 작업을 여러 번 실행해도 결과가 동일해야 함
class ProcessPaymentJob < ApplicationJob
  def perform(order_id)
    order = Order.find(order_id)

    # 이미 처리됐는지 확인!
    return if order.paid?

    Payment::Processor.call(order: order)
  end
end
```

### 4. 잡 분리 (독립적 실행)

```ruby
# ❌ BAD: 하나의 잡에 여러 작업
def perform
  task_1  # 성공
  task_2  # 실패 → 재시도 시 task_1 또 실행됨!
  task_3
end

# ✅ GOOD: 각각 별도 잡으로 분리
def perform(widget_id)
  HighPricedCheckJob.perform_later(widget_id)
  NewManufacturerCheckJob.perform_later(widget_id)
end
```

### 5. 값 고정 (재시도 대비)

```ruby
# ❌ BAD: 재시도 시 데이터가 변경되어 있을 수 있음
def perform(widget_id)
  widget = Widget.find(widget_id)
  send_email if widget.price_cents > 750_000  # 가격이 바뀌면?
end

# ✅ GOOD: 잡 생성 시점의 값을 인자로 고정
HighPricedCheckJob.perform_later(
  widget.id,
  widget.price_cents  # ← 현재 가격 고정!
)

def perform(widget_id, price_cents)
  return unless price_cents > 750_000  # 고정된 값 사용
  # ...
end
```

---

## Queue Configuration (Solid Queue)

```ruby
# Rails 8은 Solid Queue 사용
# SQLite 기반 - Redis 불필요

class ApplicationJob < ActiveJob::Base
  # 재시도 설정
  retry_on ActiveRecord::Deadlocked, wait: 5.seconds, attempts: 3
  retry_on StandardError, wait: :polynomially_longer, attempts: 3
  discard_on ActiveJob::DeserializationError
end
```

---

## 모니터링 체크리스트

잡 백엔드의 상태를 관찰할 수 있어야 합니다:

**실패한 잡 정보**
- [ ] 어떤 클래스가 실패했는지?
- [ ] 어떤 인자로 호출됐는지?
- [ ] 에러 메시지는 무엇인지?

**용량 정보**
- [ ] 대기 중인 잡이 몇 개인지?
- [ ] 어떤 종류의 잡이 대기 중인지?

**재시도 정보**
- [ ] 재시도 예정인 잡들
- [ ] 언제 재시도되는지?

---

## 멱등성 분석 방법

각 라인에서 에러가 발생하고 재시도되면 어떻게 될지 생각하세요:

```ruby
def perform(widget_id)
  widget = Widget.find(widget_id)

  if widget.price_cents > 750_000
    FinanceMailer.high_priced(widget).deliver_now  # 3번 라인
  end

  if widget.new_manufacturer?
    AdminMailer.new_manufacturer(widget).deliver_now  # 7번 라인
  end
end

# 문제점:
# - 7번 라인 실패 시 → 재무팀 이메일 이중 발송!
# - 해결: 각각 별도 잡으로 분리
```

---

## Checklist

- [ ] **ID만 전달** (객체 X)
- [ ] **서비스로 위임** (얇은 잡)
- [ ] **멱등성** (이미 처리됐는지 확인)
- [ ] **잡 분리** (여러 작업 → 별도 잡)
- [ ] **값 고정** (재시도 시 데이터 변경 대비)
- [ ] **재시도 설정** (retry_on)
- [ ] **큐 우선순위** (queue_as)
