# impl/rake_tasks - Rake Tasks 원칙

Sustainable Rails Chapter 20.2 기반 Rake Tasks 설계 원칙.

---

## 핵심 원칙: 자동화가 필요한 모든 것은 Rake Task로!

Rake Task는 **자동화 진입점**입니다.
비즈니스 로직은 반드시 Service에서 처리하세요.

---

## 언제 Rake Task를 사용하는가?

1. **정기 실행 작업** - daily batch, weekly reset 등
2. **일회성 데이터 수정** - 프로덕션 데이터 교정
3. **감사 기록 필요 작업** - 누가 언제 실행했는지 추적
4. **관리자 작업** - 역할 할당, 상태 변경

---

## Rake Task Pattern (얇은 태스크)

```ruby
# lib/tasks/reports/weekly_summary.rake
# ✅ 태스크는 진입점일 뿐 - 비즈니스 로직은 서비스에!
namespace :reports do
  desc "Generate weekly summary report"
  task weekly_summary: :environment do
    puts "Starting weekly summary generation..."

    result = Reports::WeeklySummaryService.call

    if result.success?
      puts "✅ Report generated!"
    else
      puts "❌ Failed: #{result.error}"
      exit 1
    end
  end
end

# 호출
# bin/rails reports:weekly_summary
```

---

## Core Rules (핵심 규칙)

### 1. One Task Per File (파일당 하나의 태스크)

```
# ✅ GOOD: 디렉토리 구조 = 네임스페이스
lib/tasks/
├── reports/
│   ├── weekly_summary.rake
│   └── monthly_export.rake
├── data/
│   ├── import.rake
│   └── cleanup.rake
└── production_data/
    ├── corrections/
    │   └── fix_invalid_prices.rake
    └── role_assignment/
        └── assign_admin_role.rake

# ❌ BAD: 하나의 파일에 여러 태스크
lib/tasks/
└── reports.rake  # 4개 태스크가 하나의 파일에!
```

### 2. desc 필수 (설명 작성)

```ruby
# ❌ BAD: 설명 없음
task weekly_summary: :environment do
  # ...
end

# ✅ GOOD: 명확한 설명
desc "Generate weekly summary report"
task weekly_summary: :environment do
  # ...
end

# bin/rails -T 로 확인 가능
# rake reports:weekly_summary  # Generate weekly summary report
```

### 3. 비즈니스 로직 분리 (Service로 위임)

```ruby
# ❌ BAD: 태스크에 비즈니스 로직
namespace :data do
  task check_duplicates: :environment do
    all_records = []

    # 200줄의 복잡한 중복 체크 로직...
    files.each do |file|
      # ...
    end

    # 결과 분석...
    # 리포트 생성...
  end
end

# ✅ GOOD: Service로 위임
namespace :data do
  desc "Check duplicate records across all files"
  task check_duplicates: :environment do
    puts "Checking for duplicates..."

    result = Data::DuplicateCheckerService.call

    if result.success?
      puts "✅ Check completed!"
      puts "   Duplicates found: #{result.value[:duplicate_count]}"
      puts "   Report: #{result.value[:report_path]}"
    else
      puts "❌ Failed: #{result.error}"
      exit 1
    end
  end
end
```

### 4. 인자 전달 패턴

```ruby
# 단일 인자
desc "Import data from CSV file"
task :import, [:csv_path] => :environment do |_t, args|
  unless args[:csv_path]
    puts "Usage: bin/rails data:import[path/to/file.csv]"
    exit 1
  end

  result = Data::CsvImportService.call(file_path: args[:csv_path])
  # ...
end

# 호출: bin/rails data:import[data.csv]

# 복수 인자
desc "Assign role to user"
task :assign_role, [:user_id, :role] => :environment do |_t, args|
  # ...
end

# 호출: bin/rails admin:assign_role[123,admin]
```

### 5. 에러 처리 및 종료 코드

```ruby
desc "Process daily batch"
task daily_batch: :environment do
  result = Batch::DailyProcessorService.call

  if result.success?
    puts "✅ Completed: #{result.value[:processed_count]} records"
    # exit 0 (암시적)
  else
    puts "❌ Failed: #{result.error}"
    exit 1  # 스케줄러에서 실패 감지 가능
  end
rescue StandardError => e
  puts "❌ Unexpected error: #{e.message}"
  puts e.backtrace.first(5).join("\n")
  exit 1
end
```

---

## 디렉토리 구조 예시

```
lib/tasks/
├── alerting/
│   └── check_system_health.rake
├── reports/
│   ├── weekly_summary.rake
│   └── monthly_export.rake
├── data/
│   ├── import.rake
│   ├── export.rake
│   └── cleanup.rake
└── production_data/
    ├── corrections/
    │   └── fix_invalid_prices.rake
    └── role_assignment/
        └── assign_admin_role.rake
```

---

## Checklist

- [ ] **파일당 하나의 태스크** (One Task Per File)
- [ ] **디렉토리 = 네임스페이스** (구조 일치)
- [ ] **desc 필수** (설명 작성)
- [ ] **Service로 위임** (비즈니스 로직 분리)
- [ ] **인자 검증** (필수 인자 체크)
- [ ] **에러 처리** (exit 1 반환)
- [ ] **감사 로그** (누가 언제 실행했는지)
