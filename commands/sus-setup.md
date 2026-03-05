# /sus-setup - Sustainable Rails Project Setup Agent

"Sustainable Web Development with Rails" Part 1 기반의 프로젝트 설정 에이전트.
bin/ 스크립트, 환경 설정, CI 파이프라인을 점검하고 설정합니다.

## Trigger Keywords
- sus setup, project setup
- bin/setup, bin/ci, bin/run
- environment config, dotenv
- lograge, ci pipeline

---

## Setup Checklist

### 1. Essential Scripts (bin/)

#### 1.1 bin/setup
**목적**: 새 개발자가 한 번에 프로젝트를 설정할 수 있도록 자동화

```bash
#!/usr/bin/env bash
set -e

echo "=== Installing dependencies ==="
bundle check || bundle install

echo "=== Preparing database ==="
bin/rails db:prepare  # NOT db:reset!

echo "=== Clearing logs and temp files ==="
bin/rails log:clear tmp:clear

echo "=== Restarting application server ==="
bin/rails restart

echo "=== Setup complete! ==="
```

**검사 항목**:
- [ ] `db:prepare` 사용 (db:reset 아님!)
- [ ] 멱등성 (여러 번 실행해도 안전)
- [ ] 명확한 출력 메시지
- [ ] `set -e`로 에러 시 중단

---

#### 1.2 bin/ci
**목적**: 모든 테스트와 품질 검사를 한 번에 실행

```bash
#!/usr/bin/env bash
set -e

echo "=== Running unit tests ==="
bin/rails test

echo "=== Running system tests ==="
bin/rails test:system

echo "=== Security analysis ==="
bundle exec brakeman -q -o tmp/brakeman.html

echo "=== Dependency audit ==="
bundle exec bundle-audit check --update

echo "=== All checks passed! ==="
```

**검사 항목**:
- [ ] Unit tests 실행
- [ ] System tests 실행
- [ ] Brakeman 보안 분석
- [ ] bundle-audit 의존성 검사
- [ ] 실패 시 빠른 종료 (`set -e`)

---

#### 1.3 bin/run (또는 bin/dev)
**목적**: 개발 서버를 일관되게 시작

```bash
#!/usr/bin/env bash
# PORT 환경변수 존중
bin/rails server --binding=0.0.0.0 -p ${PORT:-3000}
```

---

### 2. Environment Configuration

#### 2.1 .env 파일 구조
```
.env.development        # 개발 환경 (커밋 X)
.env.test               # 테스트 환경 (커밋 X)
.env.development.sample # 샘플 (커밋 O)
.env.test.sample        # 샘플 (커밋 O)
```

**검사 항목**:
- [ ] `.env*` 파일이 `.gitignore`에 포함
- [ ] `.env.*.sample` 파일 존재
- [ ] 필수 환경변수 문서화

#### 2.2 dotenv 설정
```ruby
# Gemfile
group :development, :test do
  gem "dotenv-rails"
end
```

---

### 3. Logging (lograge)

#### 3.1 기본 설정
```ruby
# config/initializers/lograge.rb
Rails.application.configure do
  config.lograge.enabled = Rails.env.production?
end
```

#### 3.2 향상된 설정 (request_id 포함)
```ruby
# config/initializers/lograge.rb
Rails.application.configure do
  config.lograge.enabled = true

  config.lograge.custom_options = lambda do |event|
    {
      request_id: event.payload[:request_id],
      user_id: event.payload[:user_id],
      host: event.payload[:host]
    }.compact
  end

  config.lograge.custom_payload do |controller|
    {
      host: controller.request.host,
      user_id: controller.current_user&.id
    }
  end
end
```

**검사 항목**:
- [ ] lograge gem 설치
- [ ] 프로덕션에서 활성화
- [ ] request_id 포함
- [ ] user_id 포함 (선택)

---

### 4. CI/CD Pipeline

#### 4.1 GitHub Actions 예시
```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v4

    - name: Set up Ruby
      uses: ruby/setup-ruby@v1
      with:
        bundler-cache: true

    - name: Setup database
      run: bin/rails db:prepare

    - name: Run CI
      run: bin/ci
```

**검사 항목**:
- [ ] CI 워크플로우 파일 존재
- [ ] `bin/ci` 호출 (직접 명령어 나열 아님)
- [ ] 캐싱 설정 (bundler-cache)
- [ ] 데이터베이스 설정

---

### 5. Dependency Management

#### 5.1 Gemfile 구조
```ruby
# Gemfile
source "https://rubygems.org"

ruby "3.3.0"

gem "rails", "~> 8.0.0"  # 비관적 버전 제약

# 프로덕션 gems
gem "sqlite3"
gem "puma"

group :development, :test do
  gem "dotenv-rails"
  gem "debug"
end

group :development do
  gem "web-console"
end

group :test do
  gem "capybara"
  gem "selenium-webdriver"
end

# 보안 gems
gem "brakeman", require: false
gem "bundler-audit", require: false
```

**검사 항목**:
- [ ] Ruby 버전 명시
- [ ] Rails 비관적 버전 제약 (`~>`)
- [ ] 그룹별 gem 분리
- [ ] 보안 gems 포함

---

## Execution Commands

| Command | Action |
|---------|--------|
| `/sus-setup` | 전체 체크리스트 실행 |
| `/sus-setup check` | 현재 상태만 점검 |
| `/sus-setup init` | 누락된 항목 초기화 |
| `/sus-setup fix` | 자동 수정 가능한 항목 수정 |
| `/sus-setup ci` | CI 파이프라인 설정 |
| `/sus-setup lograge` | lograge 설정 |

---

## Output Report

```
═══════════════════════════════════════════════════════════════
📦 Sustainable Rails Setup Report
═══════════════════════════════════════════════════════════════

Project: [Project Name]
Rails: 8.0.x
Ruby: 3.3.x

📁 Scripts:
   ✅ bin/setup - Idempotent, uses db:prepare
   ✅ bin/ci - 4 steps (tests, system, brakeman, audit)
   ✅ bin/dev - PORT-aware

📝 Environment:
   ✅ .gitignore includes .env*
   ⚠️ Missing .env.development.sample
   ⚠️ Missing .env.test.sample

📊 Logging:
   ✅ lograge gem installed
   ⚠️ request_id not included in logs

🔄 CI/CD:
   ❌ No .github/workflows found

📦 Dependencies:
   ✅ Ruby version specified
   ✅ Rails pessimistic constraint
   ✅ brakeman included
   ✅ bundler-audit included

═══════════════════════════════════════════════════════════════
Score: 7/11 (64%)

🔧 Recommended Actions:
1. Create .env.development.sample with required vars
2. Create .env.test.sample
3. Add request_id to lograge config
4. Setup GitHub Actions CI workflow
═══════════════════════════════════════════════════════════════
```

---

## Auto-Fix Templates

### .env.development.sample
```bash
# Database
DATABASE_URL=sqlite3:db/development.sqlite3

# Rails
RAILS_ENV=development
SECRET_KEY_BASE=your-secret-key-here

# App-specific
# EXTERNAL_API_KEY=
```

### GitHub Actions CI
```yaml
# .github/workflows/ci.yml
name: CI

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: ruby/setup-ruby@v1
      with:
        bundler-cache: true
    - run: bin/rails db:prepare
    - run: bin/ci
```
