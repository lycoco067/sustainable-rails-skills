# impl/models - Models & Database 원칙

Sustainable Rails Models 및 Database 설계 원칙.

---

## Active Record = DB Access Only

```ruby
class User < ApplicationRecord
  # ✅ 관계
  has_many :posts
  has_many :comments

  # ✅ 검증 (UX용, 데이터 무결성은 DB에서)
  validates :email, presence: true, uniqueness: true
  validates :username, format: { with: /\A[a-zA-Z0-9_]+\z/ }

  # ✅ 스코프
  scope :active, -> { where(active: true) }
  scope :recent, -> { order(created_at: :desc) }

  # ✅ 단순 데이터 접근
  def display_name
    username.presence || email.split('@').first
  end

  # ❌ NO: 비즈니스 로직 (서비스로 이동)
  # def calculate_weekly_progress
  # def send_reminder
  # def process_subscription
end
```

---

## Active Model for Non-DB

```ruby
class ContactForm
  include ActiveModel::Model
  include ActiveModel::Validations

  attr_accessor :name, :email, :message

  validates :name, :email, :message, presence: true
  validates :email, format: { with: URI::MailTo::EMAIL_REGEXP }
end
```

---

## Callbacks 최소화

```ruby
class Record < ApplicationRecord
  # ✅ OK: 데이터 정규화
  before_validation :normalize_status

  # ✅ OK: 타임스탬프 추적
  after_save :update_timestamp

  # ❌ NO: 비즈니스 로직
  # after_save :send_notification  # BAD - service로!
  # after_save :update_external_system  # BAD - service로!

  private

  def normalize_status
    self.status = status&.downcase
  end
end
```

---

## Database Constraints for Integrity

```ruby
# db/migrate/xxx_create_records.rb
class CreateRecords < ActiveRecord::Migration[8.0]
  def change
    create_table :records do |t|
      t.references :user, foreign_key: true  # FK constraint
      t.string :name, null: false           # NOT NULL
      t.string :code, null: false           # NOT NULL
      t.boolean :active, null: false, default: true
      t.integer :amount, null: false

      t.timestamps

      t.index [:user_id, :code], unique: true  # Unique constraint
    end
  end
end
```

---

## Validation vs Constraint

```ruby
# 데이터 무결성 = DB constraint
# 사용자 경험 = Rails validation

class Record < ApplicationRecord
  # Validation for UX (친절한 에러 메시지)
  validates :code, presence: { message: "코드가 필요합니다" }

  # DB constraint가 실제 무결성 보장 (migration에서)
  # null: false, foreign_key: true
end
```

---

## Checklist

### Models
- [ ] Active Record는 DB 접근만
- [ ] 비즈니스 로직은 서비스로
- [ ] 콜백은 데이터 정규화/추적만
- [ ] 스코프는 단순 쿼리만

### Database
- [ ] FK constraint 사용
- [ ] NOT NULL constraint 사용
- [ ] Unique index 사용
- [ ] Validation은 UX용 (무결성은 DB)
