# impl/views - Views & Templates 원칙

Sustainable Rails Views 설계 원칙.

---

## Single Instance Variable

```ruby
# ✅ CORRECT: 컨트롤러에서 하나의 인스턴스 변수
class WidgetsController < ApplicationController
  def show
    @widget = Widget.find(params[:id])
    # 관련 데이터는 @widget.manufacturer, @widget.ratings로 접근
  end
end

# ❌ WRONG: 여러 인스턴스 변수
def show
  @widget = Widget.find(params[:id])
  @manufacturer = @widget.manufacturer  # BAD
  @ratings = @widget.ratings            # BAD
end
```

---

## Partials with Locals

```erb
<!-- ✅ CORRECT: locals 명시 -->
<%= render partial: "widget_card", locals: { widget: @widget } %>

<!-- ❌ WRONG: 인스턴스 변수 직접 접근 -->
<%= render partial: "widget_card" %>
<!-- 파셜 내에서 @widget 직접 사용 - BAD -->
```

---

## Partial Documentation

```erb
<%# app/views/widgets/_card.html.erb %>
<%#
Widget Card Component

Parameters:
  widget::     required, Widget object to display
  show_cta::   optional, boolean (default: true)
%>

<article class="card">
  <h3><%= widget.name %></h3>
  <% if local_assigns.fetch(:show_cta, true) %>
    <%= link_to "View", widget_path(widget), class: "btn" %>
  <% end %>
</article>
```

---

## Semantic HTML

```erb
<!-- ✅ CORRECT: 시맨틱 태그 우선 -->
<article class="widget-detail">
  <header>
    <h1><%= @widget.name %></h1>
  </header>
  <section class="description">
    <p><%= @widget.description %></p>
  </section>
  <footer>
    <%= link_to "Back", widgets_path %>
  </footer>
</article>

<!-- ❌ WRONG: div 남용 -->
<div class="widget-detail">
  <div class="header">
    <div class="title"><%= @widget.name %></div>
  </div>
</div>
```

---

## ERB Only

- [ ] `.html.erb` 파일만 사용
- [ ] `.haml`, `.slim` 파일 없음
- [ ] Rails 기본 템플릿 엔진 유지

---

## Checklist

- [ ] 컨트롤러 액션당 하나의 인스턴스 변수
- [ ] 파셜 호출 시 `locals:` 명시
- [ ] 파셜 상단에 파라미터 문서화
- [ ] 시맨틱 HTML 태그 사용 (article, section, header, footer)
