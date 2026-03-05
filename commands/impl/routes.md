# impl/routes - Routes 원칙

Sustainable Rails Routes 설계 원칙.

---

## Canonical Routes 원칙

```ruby
# ✅ ALWAYS: resources with :only
resources :widgets, only: [:index, :show, :create, :update, :destroy]

# ✅ ALWAYS: 사용하는 액션만 명시
resources :users, only: [:show, :edit, :update]

# ❌ NEVER: 모든 7개 액션 노출
resources :users  # BAD - 불필요한 라우트 노출
```

---

## Custom Actions 금지

```ruby
# ❌ NEVER: 커스텀 액션
resources :widgets do
  member do
    post :archive      # BAD!
    post :publish      # BAD!
  end
end

# ✅ ALWAYS: 새 리소스로 분리
resources :widgets, only: [:show]
resources :widget_archives, only: [:create]
resources :widget_publications, only: [:create]
```

---

## Nesting 제한

```ruby
# ✅ OK: 1단계 중첩
resources :users do
  resources :posts, only: [:index, :create]
end

# ❌ NEVER: 2단계 이상 중첩
resources :users do
  resources :posts do
    resources :comments  # TOO DEEP!
  end
end

# ✅ BETTER: shallow 또는 별도 리소스
resources :users do
  resources :posts, only: [:index, :create], shallow: true
end
resources :post_comments, only: [:create]
```

---

## API Versioning

```ruby
# ✅ URL versioning
namespace :api do
  namespace :v1 do
    resources :widgets, only: [:index, :show]
  end
end

# Path: /api/v1/widgets
# Controller: Api::V1::WidgetsController
```

---

## Vanity URLs

```ruby
# ✅ redirect 사용 (중복 콘텐츠 방지)
get "/u/:username", to: redirect { |params, req|
  user = User.find_by(username: params[:username])
  user ? "/users/#{user.id}" : "/404"
}

# ❌ NEVER: 직접 매핑
get "/u/:username", to: "users#show"  # BAD - 중복 콘텐츠
```

---

## Checklist

- [ ] 모든 `resources`에 `:only` 또는 `:except` 명시
- [ ] 커스텀 액션 없음 (member/collection do 내)
- [ ] 중첩 깊이 1단계 이하
- [ ] API 라우트는 `/api/v1/` 네임스페이스
