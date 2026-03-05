# /sustainable-check - Sustainable Rails 원칙 검사 에이전트

"Sustainable Web Development with Rails" 책의 원칙을 기반으로 Rails 프로젝트를 점검하는 에이전트입니다.

## Trigger Keywords
- sustainable check, 지속가능성 검사
- rails 원칙 검사, 코드 품질 점검
- sustainable rails

---

## 검사 영역

### 1. Routes 검사 (라우팅)

#### 1.1 Canonical Routes 사용 여부
```ruby
# ✅ Good: resources 사용
resources :widgets, only: [:index, :show, :create]

# ❌ Bad: get/post 직접 사용
get "/widgets/:id", to: "widgets#show"
post "/widgets", to: "widgets#create"
```

**검사 항목**:
- [ ] `config/routes.rb`에서 `resources` 또는 `resource` 사용 비율
- [ ] `get`, `post`, `patch`, `put`, `delete` 직접 사용 시 정당한 이유 있는지
- [ ] 각 resources에 `:only` 또는 `:except` 명시되어 있는지

#### 1.2 사용하지 않는 라우트 여부
```ruby
# ✅ Good: 필요한 액션만 정의
resources :users, only: [:show, :edit, :update]

# ❌ Bad: 모든 7개 액션 노출
resources :users
```

#### 1.3 커스텀 액션 남용 여부
```ruby
# ❌ Bad: 커스텀 액션
resources :widgets do
  post :update_rating
  post :archive
end

# ✅ Good: 새 리소스로 분리
resources :widgets, only: [:show]
resources :widget_ratings, only: [:create]
resources :widget_archives, only: [:create]
```

#### 1.4 중첩 라우트 깊이
```ruby
# ❌ Bad: 2단계 이상 중첩
resources :manufacturers do
  resources :widgets do
    resources :ratings  # 너무 깊음!
  end
end

# ✅ Good: 1단계 또는 shallow
resources :widgets, only: [:show]
resources :widget_ratings, only: [:create]
```

---

### 2. HTML Templates 검사 (뷰)

#### 2.1 시맨틱 HTML 사용
```erb
<!-- ✅ Good: 시맨틱 태그 -->
<article>
  <header><h1><%= @post.title %></h1></header>
  <section><%= @post.body %></section>
</article>

<!-- ❌ Bad: div 남용 -->
<div class="post">
  <div class="title"><%= @post.title %></div>
  <div class="body"><%= @post.body %></div>
</div>
```

#### 2.2 인스턴스 변수 개수
```ruby
# ✅ Good: 하나의 인스턴스 변수
def show
  @widget = Widget.find(params[:id])
end

# ❌ Bad: 여러 인스턴스 변수
def show
  @widget = Widget.find(params[:id])
  @manufacturer = @widget.manufacturer
  @ratings = @widget.ratings
  @similar_widgets = Widget.similar_to(@widget)
end
```

#### 2.3 파셜 사용 원칙
```erb
<!-- ✅ Good: locals 사용 -->
<%= render partial: "widget", locals: { widget: @widget } %>

<!-- ❌ Bad: 인스턴스 변수 직접 사용 -->
<%= render partial: "widget" %>  <!-- @widget 직접 접근 -->
```

---

### 3. Helpers 검사

#### 3.1 도메인 로직 분리
```ruby
# ❌ Bad: 헬퍼에 도메인 로직
def formatted_widget_id(widget)
  widget.id.to_s.insert(-3, '.')  # 도메인 로직!
end

# ✅ Good: 모델에 도메인 로직
class Widget < ApplicationRecord
  def formatted_id
    id.to_s.insert(-3, '.')
  end
end
```

#### 3.2 안전한 마크업 생성
```ruby
# ✅ Good: Rails API 사용
def styled_id(id)
  content_tag(:span, id, class: "mono")
end

# ❌ Bad: 문자열 직접 조합 + html_safe
def styled_id(id)
  "<span class='mono'>#{id}</span>".html_safe
end
```

---

## 실행 워크플로우

### Phase 1: 자동 스캔
```yaml
scan_targets:
  - config/routes.rb
  - app/controllers/**/*.rb
  - app/views/**/*.html.erb
  - app/helpers/**/*.rb
  - app/models/**/*.rb
```

### Phase 2: 점수 산출

| 영역 | 가중치 | 검사 항목 수 |
|------|--------|-------------|
| Routes | 30% | 8개 |
| Templates | 40% | 10개 |
| Helpers | 30% | 6개 |

### Phase 3: 리포트 생성

```markdown
# Sustainable Rails 검사 리포트

## 종합 점수: 85/100 ⭐⭐⭐⭐

### Routes (28/30)
✅ resources 사용률: 95%
✅ :only 명시율: 100%
⚠️ 커스텀 액션 2개 발견 (분리 권장)

### Templates (35/40)
✅ 시맨틱 HTML 사용
⚠️ 인스턴스 변수 2개 이상인 액션: 3개
❌ 파셜에서 @변수 직접 사용: 5개

### Helpers (22/30)
✅ content_tag 사용률: 80%
⚠️ html_safe 직접 호출: 3개
❌ 헬퍼 테스트 미존재

## 개선 권장사항
1. widgets_controller.rb:45 - @manufacturer 제거
2. _sidebar.html.erb:12 - locals 사용으로 변경
3. application_helper.rb:23 - html_safe 제거
```

---

## 명령어 사용법

```bash
# 전체 검사
/sustainable-check

# 특정 영역만 검사
/sustainable-check --routes
/sustainable-check --templates
/sustainable-check --helpers

# 특정 파일/폴더만 검사
/sustainable-check app/controllers/widgets_controller.rb
/sustainable-check app/views/widgets/

# 자동 수정 제안
/sustainable-check --fix

# 리포트 저장
/sustainable-check --report docs/sustainable_report.md
```

---

## 검사 기준 상세

### Routes 세부 기준

| 항목 | 점수 | 기준 |
|------|------|------|
| resources 사용률 | 10점 | 90%+ = 10, 70%+ = 7, 50%+ = 4 |
| :only 명시율 | 5점 | 100% = 5, 80%+ = 3, 60%+ = 1 |
| 커스텀 액션 수 | 5점 | 0개 = 5, 1-2개 = 3, 3개+ = 0 |
| 중첩 깊이 | 5점 | 1단계 = 5, 2단계 = 2, 3단계+ = 0 |
| Vanity URL 처리 | 5점 | redirect 사용 = 5, 직접 매핑 = 2 |

### Templates 세부 기준

| 항목 | 점수 | 기준 |
|------|------|------|
| 시맨틱 HTML | 10점 | div 남용 없음 = 10, 일부 = 5 |
| 인스턴스 변수 | 10점 | 모두 1개 = 10, 일부 2개 = 5 |
| 파셜 locals | 10점 | 100% = 10, 80%+ = 6, 50%+ = 3 |
| ERB 사용 | 5점 | 100% ERB = 5, 혼용 = 0 |
| 파셜 문서화 | 5점 | 주석 있음 = 5, 없음 = 0 |

### Helpers 세부 기준

| 항목 | 점수 | 기준 |
|------|------|------|
| 도메인 분리 | 10점 | 완벽 = 10, 일부 혼재 = 5 |
| Rails API 사용 | 10점 | 100% = 10, 80%+ = 6 |
| html_safe 최소화 | 5점 | 0개 = 5, 1-2개 = 3, 3개+ = 0 |
| 테스트 존재 | 5점 | 있음 = 5, 없음 = 0 |

---

## 출력 예시

```
═══════════════════════════════════════════════════════════════
🔍 Sustainable Rails 검사 시작
   프로젝트: [Project Name]
   검사 시간: 2024-01-15 14:30:22
═══════════════════════════════════════════════════════════════

📁 스캔 중...
   - config/routes.rb ✓
   - app/controllers/ (12 files) ✓
   - app/views/ (45 files) ✓
   - app/helpers/ (3 files) ✓

═══════════════════════════════════════════════════════════════
📊 ROUTES 검사 결과 (28/30)
═══════════════════════════════════════════════════════════════

✅ resources 사용률: 95% (19/20 라우트)
✅ :only 명시: 100%
⚠️ 커스텀 액션 발견: 2개
   └─ config/routes.rb:34 - post :archive (→ archives 리소스 분리 권장)
✅ 중첩 깊이: 1단계 (권장 범위 내)

═══════════════════════════════════════════════════════════════
📊 TEMPLATES 검사 결과 (32/40)
═══════════════════════════════════════════════════════════════

✅ 시맨틱 HTML: 양호
⚠️ 인스턴스 변수 2개 이상: 3개 액션
❌ 파셜에서 @변수 직접 사용: 5개
✅ ERB 사용: 100%

═══════════════════════════════════════════════════════════════
📊 HELPERS 검사 결과 (22/30)
═══════════════════════════════════════════════════════════════

✅ 도메인 로직 분리: 양호
⚠️ html_safe 직접 호출: 3개
✅ Rails API 사용: 85%
❌ 헬퍼 테스트: 미존재

═══════════════════════════════════════════════════════════════
📈 종합 결과
═══════════════════════════════════════════════════════════════

   Routes:    ████████░░ 28/30 (93%)
   Templates: ████████░░ 32/40 (80%)
   Helpers:   ███████░░░ 22/30 (73%)
   ─────────────────────────────────
   TOTAL:     ████████░░ 82/100 ⭐⭐⭐⭐

📝 우선 개선 항목:
   1. 파셜의 @변수 → locals 변환 (영향도: 높음)
   2. 헬퍼 테스트 추가 (영향도: 중간)
   3. html_safe → content_tag 변환 (영향도: 중간)

═══════════════════════════════════════════════════════════════
```
