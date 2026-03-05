# /sus-implement - Implementation Orchestrator

"Sustainable Web Development with Rails" Part 2 기반의 구현 오케스트레이터.
서브에이전트를 **병렬로** 실행하여 빠른 구현을 지원합니다.

## Trigger Keywords
- sus implement, build feature
- routes, views, helpers
- models, services, controllers

---

## Sub-Agent Routing

| Command | Sub-agents | Parallel |
|---------|-----------|----------|
| `/sus-implement <feature>` | 분석 후 필요한 것들 | ✅ |
| `/sus-implement --routes` | impl/routes.md | - |
| `/sus-implement --views` | impl/views.md + impl/helpers.md | ✅ |
| `/sus-implement --backend` | impl/models.md + impl/services.md + impl/controllers.md | ✅ |
| `/sus-implement --frontend` | impl/frontend.md + impl/views.md | ✅ |
| `/sus-implement --model` | impl/models.md | - |
| `/sus-implement --service` | impl/services.md | - |
| `/sus-implement --controller` | impl/controllers.md | - |
| `/sus-implement --job` | impl/jobs.md | - |

---

## Sub-Agent Files

```
impl/
├── routes.md       # Routes 원칙
├── views.md        # Views & Templates
├── helpers.md      # Helpers
├── frontend.md     # CSS + JavaScript
├── models.md       # Models + Database
├── services.md     # Business Logic
├── controllers.md  # Controllers
├── jobs.md         # Background Jobs
├── mailers.md      # Email
└── rake_tasks.md   # Rake Tasks
```

---

## Parallel Execution Pattern

`/sus-implement <feature>` 실행 시:

```
1. 요청 분석: 어떤 레이어가 필요한가?
   - Routes? Models? Services? Controllers? Views?

2. Phase 1 - 독립적인 작업들 (병렬):
   ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
   │impl/routes  │ │impl/models  │ │impl/services│
   └─────────────┘ └─────────────┘ └─────────────┘
         ↓               ↓               ↓
         └───────────────┼───────────────┘
                         ↓
3. Phase 2 - 의존성 있는 작업들 (순차):
                  ┌─────────────┐
                  │impl/controllers│
                  └──────┬──────┘
                         ↓
                  ┌─────────────┐
                  │impl/views   │
                  └─────────────┘
```

---

## Feature Analysis

요청을 분석하여 필요한 서브에이전트 결정:

| Feature Type | Required Sub-agents |
|--------------|---------------------|
| CRUD 기능 | routes + models + services + controllers + views |
| API 엔드포인트 | routes + services + controllers |
| 백그라운드 작업 | jobs + services |
| UI 변경 | views + helpers + frontend |
| 비즈니스 로직 | services + models |

---

## Execution Example

```
User: /sus-implement 리더보드 기능

Orchestrator:
  📊 분석 결과:
  - Routes: resources :leaderboards
  - Models: Leaderboard (또는 기존 모델 활용)
  - Services: Leaderboard::Calculator
  - Controllers: LeaderboardsController
  - Views: leaderboards/index.html.erb

  🚀 Phase 1 (병렬 실행):
  - Task: impl/routes.md → "리더보드 라우트 설계"
  - Task: impl/models.md → "리더보드 모델 설계"
  - Task: impl/services.md → "Leaderboard::Calculator 서비스"

  ⏳ Phase 1 완료 대기...

  🚀 Phase 2 (순차 실행):
  - Task: impl/controllers.md → "LeaderboardsController"
  - Task: impl/views.md → "리더보드 뷰"

  ✅ 완료!
```

---

## Quick Commands

### Single Layer
```bash
/sus-implement --routes       # Routes만
/sus-implement --model        # Model만
/sus-implement --service      # Service만
/sus-implement --controller   # Controller만
/sus-implement --views        # Views만
/sus-implement --job          # Job만
```

### Multi Layer (Parallel)
```bash
/sus-implement --backend      # models + services + controllers
/sus-implement --frontend     # frontend + views + helpers
```

### Full Feature
```bash
/sus-implement <feature>      # 전체 분석 후 병렬/순차 실행
```

---

## Integration

- `/sus check` → 구현 후 원칙 준수 검사
- `/sus-test` → 구현 후 테스트 작성
- `/refactor-to-services` → Fat model 발견 시 서비스 추출
