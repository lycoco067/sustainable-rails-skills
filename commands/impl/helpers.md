# impl/helpers - Helpers 원칙

Sustainable Rails Helpers 설계 원칙.

---

## Domain Logic → Models

```ruby
# ❌ WRONG: 헬퍼에 도메인 로직
module ProgressHelper
  def formatted_score(progress)
    "#{(progress.correct_count.to_f / progress.total_count * 100).round}%"
  end
end

# ✅ CORRECT: 모델에 도메인 로직
class UserProgress < ApplicationRecord
  def score_percentage
    return 0 if total_count.zero?
    (correct_count.to_f / total_count * 100).round
  end
end

# 헬퍼는 마크업만
module ProgressHelper
  def score_badge(progress)
    content_tag(:span, "#{progress.score_percentage}%", class: "score-badge")
  end
end
```

---

## Safe Markup Generation

```ruby
# ✅ CORRECT: Rails API 사용
def star_rating(count)
  content_tag(:div, class: "stars") do
    safe_join(count.times.map { content_tag(:span, "⭐") })
  end
end

# ❌ WRONG: html_safe 남용
def star_rating(count)
  "<div class='stars'>#{'⭐' * count}</div>".html_safe  # XSS 위험!
end
```

---

## Helper 테스트

```ruby
# test/helpers/progress_helper_test.rb
class ProgressHelperTest < ActionView::TestCase
  test "score_badge renders span with percentage" do
    progress = OpenStruct.new(score_percentage: 85)
    result = score_badge(progress)

    assert_match /<span.*85%.*<\/span>/, result
    assert result.html_safe?
  end
end
```

---

## Checklist

- [ ] 도메인 로직은 모델에, 헬퍼는 마크업만
- [ ] `content_tag`, `safe_join` 등 Rails API 사용
- [ ] `html_safe` 직접 호출 금지
- [ ] 헬퍼 테스트 작성
