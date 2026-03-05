# impl/frontend - CSS & JavaScript 원칙

Sustainable Rails Frontend 설계 원칙.
CSS와 JavaScript를 통합하여 다룹니다.

---

## CSS (Design System)

### 디자인 시스템 사용
```erb
<!-- ✅ CORRECT: 디자인 시스템 클래스 사용 -->
<button class="btn btn-primary">Submit</button>
<div class="card">Content</div>
<input class="form-input" placeholder="Enter...">

<!-- ❌ WRONG: 커스텀 인라인 스타일 -->
<button style="background: blue; border: 2px solid black;">Submit</button>
```

### Color Tokens
```erb
<!-- 정의된 색상만 사용 (theme.css 또는 tailwind.config.js) -->
bg-primary    <!-- Primary -->
bg-success    <!-- Success -->
bg-danger     <!-- Warning/Error -->
bg-info       <!-- Info -->
```

### Consistent Styling
```
✅ ALWAYS: 프로젝트 디자인 시스템 따르기
✅ ALWAYS: CSS 변수 또는 Tailwind 토큰 사용
❌ NEVER: 하드코딩된 색상값
❌ NEVER: 일관성 없는 스타일링
```

참조: 프로젝트의 `docs/DESIGN_TOKENS.md` 또는 `theme.css`

---

## JavaScript (Stimulus Focus)

### Minimize JavaScript
```
원칙: 서버 렌더링 우선, JS는 점진적 향상만
- 페이지 렌더링: 서버 (ERB)
- 폼 처리: 서버 (Turbo)
- 인터랙션: Stimulus
- 클라이언트 렌더링: 최소화
```

### Stimulus Controller Pattern
```javascript
// app/javascript/controllers/form_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["input", "feedback", "submitButton"]
  static values = {
    endpoint: String,
    minLength: { type: Number, default: 3 }
  }

  connect() {
    this.validate()
  }

  validate(event) {
    const value = this.inputTarget.value
    const isValid = value.length >= this.minLengthValue

    this.submitButtonTarget.disabled = !isValid
  }

  async submit(event) {
    event.preventDefault()

    const response = await fetch(this.endpointValue, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ value: this.inputTarget.value })
    })
  }
}
```

### Stimulus 계층 예시
```
form_controller.js           # 폼 오케스트레이터
├── validation, submission 관리
└── field/*_controller.js    # 타입별 핸들러
    ├── text_field_controller.js
    ├── select_field_controller.js
    └── file_upload_controller.js
```

### JavaScript 테스트 (Vitest)
```javascript
// app/javascript/controllers/form_controller.test.js
import { describe, it, expect, beforeEach } from 'vitest'

describe('FormController', () => {
  it('disables submit button when input is too short', () => {
    // ...
  })
})
```

---

## Checklist

### CSS
- [ ] 디자인 시스템 클래스 사용
- [ ] 정의된 색상 토큰만 사용
- [ ] 일관된 스타일링 패턴

### JavaScript
- [ ] 서버 렌더링 우선
- [ ] Stimulus 컨트롤러 패턴
- [ ] Vitest로 테스트
