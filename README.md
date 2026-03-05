# Sustainable Rails Skills for Claude Code

> Claude Code skills based on ["Sustainable Web Development with Rails"](https://sustainable-rails.com/) by David Bryant Copeland

A comprehensive set of Claude Code commands that help you build Rails applications following sustainable development principles. These skills guide you through routes, models, services, controllers, views, testing, and operations with best practices baked in.

## Features

- **Principle-Driven Development** - Enforces sustainable Rails patterns automatically
- **Parallel Execution** - Sub-agents run concurrently for faster implementation
- **Testing Pyramid** - System, Unit, Service, and JavaScript test templates
- **Service Layer Pattern** - ApplicationService with Result objects
- **Operations Ready** - Auth, API design, logging, monitoring, and error handling

## Quick Start

```bash
# Clone to your project
git clone https://github.com/YOUR_USERNAME/sustainable-rails-skills.git /tmp/sus-skills

# Copy to your Rails project
mkdir -p .claude/commands
cp -r /tmp/sus-skills/commands .claude/commands/sus

# Clean up
rm -rf /tmp/sus-skills
```

## Commands

### Main Router

```bash
/sus                      # Interactive mode - analyzes your request
/sus check                # Run principle compliance check
/sus setup                # Project setup verification
/sus implement <feature>  # Feature implementation guide
/sus test                 # Testing strategy and templates
/sus ops                  # Operations configuration
```

### Direct Commands

| Command | Description |
|---------|-------------|
| `/sus-setup` | Check bin/scripts, env vars, logging, CI/CD |
| `/sus-implement` | Orchestrate implementation across layers |
| `/sus-test` | Generate tests following Testing Pyramid |
| `/sus-ops` | Auth, API, logging, monitoring setup |
| `/sustainable-check` | Full codebase principle audit |
| `/refactor-to-services` | Extract business logic to services |

### Implementation Flags

```bash
/sus-implement <feature>     # Full analysis + parallel execution
/sus-implement --backend     # models + services + controllers
/sus-implement --frontend    # views + helpers + frontend (CSS/JS)
/sus-implement --routes      # Routes only
/sus-implement --model       # Model only
/sus-implement --service     # Service only
```

### Testing Commands

```bash
/sus-test system             # System test template
/sus-test unit User          # Unit test for model
/sus-test service OrderCreator  # Service test
/sus-test js quiz_controller # Vitest for Stimulus
/sus-test scenario           # Run + auto-fix errors
```

## Directory Structure

```
.claude/commands/sus/
├── sus.md                    # Main router
├── sus-setup.md              # Project setup
├── sus-implement.md          # Implementation orchestrator
├── sus-test.md               # Testing agent
├── sus-ops.md                # Operations
├── sustainable-check.md      # Principle checker
├── impl/                     # Implementation principles
│   ├── routes.md             # Canonical routes, :only, no custom actions
│   ├── models.md             # DB access only, constraints
│   ├── services.md           # Stateless, Result objects
│   ├── controllers.md        # Thin, parameter conversion
│   ├── views.md              # Single @var, locals, semantic HTML
│   ├── helpers.md            # Markup only, safe generation
│   ├── frontend.md           # Stimulus, CSS tokens
│   ├── jobs.md               # ID only, idempotent, service delegation
│   ├── mailers.md            # Format only, deliver_later
│   └── rake_tasks.md         # One task per file, service delegation
└── sub_made/
    └── refactor-to-services.md  # Fat model → services
```

## Core Principles

### Routes
```ruby
# Always use resources with :only
resources :widgets, only: [:index, :show, :create]

# No custom actions - create new resources instead
resources :widget_archives, only: [:create]  # not post :archive
```

### Models
```ruby
class Widget < ApplicationRecord
  # DB relationships, validations, scopes ONLY
  has_many :parts
  validates :name, presence: true
  scope :active, -> { where(active: true) }

  # NO business logic here!
end
```

### Services
```ruby
class Order::Creator < ApplicationService
  def initialize(user:, items:)
    @user = user
    @items = items
  end

  def call
    order = Order.create!(user: @user, items: @items)
    success(order: order)
  rescue => e
    failure(e.message)
  end
end
```

### Controllers
```ruby
class OrdersController < ApplicationController
  def create
    result = Order::Creator.call(user: current_user, items: params[:items])

    if result.success?
      redirect_to order_path(result.value[:order])
    else
      render :new, alert: result.error
    end
  end
end
```

### Views
```erb
<%# Always use locals with partials %>
<%= render partial: "widget_card", locals: { widget: @widget } %>

<%# Semantic HTML %>
<article>
  <header><h1><%= @widget.name %></h1></header>
  <section><%= @widget.description %></section>
</article>
```

### Testing
```ruby
# Use data-testid for reliable selectors
find("[data-testid='submit-button']").click

# System tests for critical user flows only
# Unit tests for edge cases and calculations
# Service tests for business logic
```

## Prerequisites

### ApplicationService Base Class

Add this to your Rails project:

```ruby
# app/services/application_service.rb
class ApplicationService
  def self.call(**args)
    new(**args).call
  end

  private

  def success(value = {})
    Result.new(success: true, value: value)
  end

  def failure(error)
    Result.new(success: false, error: error)
  end

  Result = Struct.new(:success, :value, :error, keyword_init: true) do
    def success? = success
    def failure? = !success
  end
end
```

## Customization

Each `.md` file can be customized for your project:

1. **Design System** - Update `impl/frontend.md` with your CSS classes
2. **Testing Selectors** - Modify `sus-test.md` data-testid patterns
3. **Service Naming** - Adjust conventions in `impl/services.md`
4. **Project Structure** - Update paths in `sus-setup.md`

## References

- [Sustainable Web Development with Rails](https://sustainable-rails.com/) - The book these skills are based on
- [Rails Guides](https://guides.rubyonrails.org/) - Official Rails documentation
- [Stimulus Handbook](https://stimulus.hotwired.dev/handbook/introduction) - Hotwire Stimulus

## Contributing

1. Fork this repository
2. Create your feature branch (`git checkout -b feature/amazing-skill`)
3. Commit your changes (`git commit -m 'Add amazing skill'`)
4. Push to the branch (`git push origin feature/amazing-skill`)
5. Open a Pull Request

## License

MIT License - feel free to use in your projects!

---

**Built with Claude Code** - Making sustainable Rails development accessible to everyone.
