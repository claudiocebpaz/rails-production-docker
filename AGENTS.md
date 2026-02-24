# AGENTS.md - Developer Guide for AI Agents

> **⚠️ IMPORTANTE: Este proyecto es 100% Dockerizado - NADA se ejecuta en el host**
> 
> **NUNCA** ejecutes comandos directamente en tu máquina local.
> **NUNCA** uses `bundle`, `rails`, `yarn`, `rake`, `ruby`, `node` directamente en tu terminal.
> **SIEMPRE** usa el wrapper `./run` para ejecutar TODO dentro de los contenedores Docker.
> 
> El entorno local NO tiene Ruby, Rails, Node.js ni ninguna dependencia instalada.
> Esto garantiza consistencia entre desarrollo, CI y producción.
> 
> **参考**: https://guides.rubyonrails.org/generators.html

This file provides context and guidelines for AI agents working on this Rails + Docker project.

## Project Overview

- **Rails Version**: 8.1.2
- **Ruby Version**: 4.0.1
- **App Name**: hello
- **Docker**: **MANDATORY** (100% containerized - no local dependencies)
- **Database**: PostgreSQL (via Docker)
- **Cache/Background Jobs**: Redis + Sidekiq (via Docker)

## Tech Stack

| Layer | Technology |
|-------|------------|
| Backend | Rails 8, Ruby 4.0.1, Sidekiq |
| Database | PostgreSQL |
| Cache | Redis |
| Frontend | esbuild, TailwindCSS 4.x, Hotwire Turbo, StimulusJS |
| Testing | Minitest, Capybara, Selenium |

---

## Build, Lint & Test Commands

### Running the Application

```bash
# Start all services (PostgreSQL, Redis, Rails)
docker compose up --build

# Setup database
./run rails db:setup
```

### Running Tests

```bash
# Run all tests
./run test

# Run tests with JS/CSS build first (for CI)
./run test -b

# Run a single test file
./run rails test test/controllers/pages_controller_test.rb

# Run a single test method
./run rails test test/controllers/pages_controller_test.rb -n test_home_page
```

### Code Quality & Formatting

```bash
# Format Ruby code (Rubocop)
./run format

# Auto-correct formatting issues
./run format --auto-correct
# or shorthand
./run format -a

# Run all quality checks (Dockerfile, shell, Ruby)
./run quality

# Lint Dockerfile
./run lint:dockerfile

# Lint shell scripts
./run lint:shell

# Format shell scripts
./run format:shell

# Check for outdated gems
./run bundle:outdated

# Check for outdated yarn packages
./run yarn:outdated
```

### Asset Building

```bash
# Build JS assets
./run yarn build

# Build CSS assets
./run yarn build:css

# Build all assets (via package.json scripts, dentro del contenedor)
./run yarn build
```

### Rails Commands

```bash
# Any Rails command via Docker
./run rails <command>

# Example: generate a controller
./run rails g controller Products

# Example: run migrations
./run rails db:migrate
```

### Rails Generators

**SIEMPRE usa generadores de Rails** en lugar de crear archivos manualmente. Los generadores:

- Crean la estructura correcta automáticamente
- Generan tests y fixtures
- Configuran rutas y dependencias
- **SE EJECUTAN DENTRO DEL CONTENEDOR Docker** (nunca en el host)

```bash
# ⚠️ CORRECTO - Ejecutar dentro del contenedor
./run rails g resource Product name:string price:decimal description:text

# ❌ INCORRECTO - No ejecutar nunca en el host
# rails g resource Product name:string price:decimal description:text

# Generar un recurso completo (modelo, controlador, vistas, migración)
./run rails g resource Product name:string price:decimal description:text

# Generar solo modelo
./run rails g model User email:string name:string

# Generar solo controlador
./run rails g controller Products

# Generar migración
./run rails g migration AddFieldsToUsers

# Generar job
./run rails g job ProcessOrder

# Generar mailer
./run rails g mailer UserMailer

# Revertir el último generador
./run rails d resource Product
```

**Ver también**: https://guides.rubyonrails.org/generators.html

### Dependency Management

```bash
# Install all dependencies
./run deps:install

# Install without rebuilding Docker image
./run deps:install --no-build
```

---

## Code Style Guidelines

### Ruby Style (Rubocop - Omakase)

This project uses the [rubocop-rails-omakase](https://github.com/rails/rubocop-rails-omakase/) gem, which applies Rails' default "omakase" style. Key conventions:

- **2-space indentation** (no tabs)
- **Single quotes** for strings unless interpolation is needed
- **No trailing whitespace**
- **Line length**: Up to 120 characters (Rubocop default)
- **Spaces after commas**: `[a, b, c]` not `[a,b,c]`
- **No frozen string literal comments** (Ruby 3+ default)

### File Organization

```
app/
  controllers/    # Controllers (e.g., PagesController)
  models/         # Models (e.g., User)
  views/          # ERB templates
  helpers/        # View helpers
  jobs/           # Sidekiq jobs
  channels/      # Action Cable channels
  javascript/     # esbuild entry points
  assets/         # Stylesheets, images
config/
  initializers/   # Rails config
  environments/   # Environment-specific config
lib/
  tasks/          # Rake tasks
test/             # Minitest tests
```

### Naming Conventions

| Type | Convention | Example |
|------|------------|---------|
| Models | PascalCase | `User`, `OrderItem` |
| Controllers | PascalCase + Controller | `PagesController` |
| Views | snake_case + action | `pages/home.html.erb` |
| Database tables | snake_case plural | `users`, `order_items` |
| Methods | snake_case | `def user_name`, `def order_total` |
| Variables | snake_case | `@user`, `@order_items` |
| Constants | SCREAMING_SNAKE_CASE | `MAX_RETRIES`, `DEFAULT_LIMIT` |
| Routes | snake_case | `resources :users` |

### Class Structure

```ruby
class PagesController < ApplicationController
  # 1. Constants (if any)
  MAX_ITEMS = 10

  # 2. Public methods
  def home
    # Action methods first
  end

  # 3. Private methods
  private

  def some_helper_method
    # ...
  end
end
```

### Error Handling

- Use `fail` (not `raise`) to re-raise exceptions
- Use custom error classes for domain-specific errors
- Always rescue specific exceptions, not broad `StandardError`

```ruby
# Good
begin
  external_service.call
rescue ExternalService::TimeoutError => e
  Rails.logger.error("Service timeout: #{e.message}")
  render json: { error: "Service unavailable" }, status: :service_unavailable
end
```

### Database (ActiveRecord)

- Use `find_by` for single records (returns `nil` if not found)
- Use `find` only when expecting the record to exist (raises `ActiveRecord::RecordNotFound`)
- Scope commonly used queries in the model
- Use `where.not` for negations

```ruby
# Good
scope :active, -> { where(active: true) }
User.active.find_by(email: params[:email])

# Avoid
User.where("active = ?", true)
```

### Views (ERB)

- Use helper methods for repeated logic
- Use partials for reusable view components
- Avoid complex logic in views; move to helpers or presenters
- Use Turbo Frames/Streams for SPA-like behavior

### JavaScript

- Entry point: `app/javascript/application.js`
- Use StimulusJS for interactivity
- Use Turbo Drive for page navigation
- Use TailwindCSS classes for styling

### CSS

- Use TailwindCSS utility classes
- Entry point: `app/assets/stylesheets/application.tailwind.css`
- Build via: `./run yarn build:css`

---

## Docker Development

### Container Services

| Service | Port | Description |
|---------|------|-------------|
| web | 8000 | Rails application |
| postgres | 5432 | PostgreSQL database |
| redis | 6379 | Redis cache |
| sidekiq | - | Background job worker |

### Useful Docker Commands

```bash
# View logs
docker compose logs -f web

# Access Rails console
./run rails c

# Access shell in container
./run shell

# Connect to PostgreSQL
./run psql

# Connect to Redis
./run redis-cli

# Rebuild and start
docker compose up --build

# Stop all services
docker compose down
```

---

## Testing Guidelines

### Test Framework

- **Framework**: Minitest
- **System Tests**: Capybara + Selenium (headless Chrome)
- **Run**: `./run test`

### Running Specific Tests

```bash
# Single file
./run rails test test/controllers/pages_controller_test.rb

# Single test method
./run rails test test/controllers/pages_controller_test.rb -n test_home_page

# By pattern
./run rails test test/controllers/*_test.rb
```

### Test Patterns

```ruby
class PagesControllerTest < ActionDispatch::IntegrationTest
  test "home page returns success" do
    get root_url
    assert_response :success
  end
end
```

### System Tests

- Located in `test/application_system_test_case.rb`
- Use Capybara for browser automation

---

## Environment Variables

Copy `.env.example` to `.env` and configure:

| Variable | Description |
|----------|-------------|
| `POSTGRES_USER` | Database username |
| `POSTGRES_PASSWORD` | Database password |
| `DATABASE_URL` | Full database connection string |
| `REDIS_URL` | Redis connection string |
| `DOCKER_WEB_PORT` | Port for Rails server (default: 8000) |
| `RAILS_ENV` | Environment (development, test, production) |

---

## CI/CD

GitHub Actions workflow: `.github/workflows/ci.yml`

The CI pipeline:
1. Lints Dockerfile and shell scripts
2. Formats shell scripts
3. Runs Rubocop checks
4. Sets up Docker environment
5. Creates test database
6. Runs tests with asset build

---

## Additional Resources

- [Rails Guides](https://guides.rubyonrails.org/)
- [Hotwire Turbo](https://hotwired.dev/)
- [StimulusJS](https://stimulus.hotwired.dev/)
- [TailwindCSS](https://tailwindcss.com/)
- [Docker Best Practices](https://docs.docker.com/)
