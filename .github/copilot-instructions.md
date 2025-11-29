# GitHub Community Discussions Automation

This repository manages the **GitHub Community Discussions** forum through automated workflows that handle discussion lifecycle, labeling, and incident management.

## Architecture Overview

This is a **Ruby-based GitHub Actions automation system** that:
- Monitors and manages community discussions via GraphQL API
- Auto-labels, comments on, and closes discussions based on lifecycle rules
- Handles incident discussions with status tracking (open/update/resolve/close)
- Uses scheduled and event-triggered workflows

### Core Components

- **`.github/workflows/`** - GitHub Actions workflows (event-based and scheduled)
- **`.github/actions/`** - Executable Ruby scripts for automation tasks
- **`.github/lib/`** - Shared Ruby classes (`GitHub`, `Discussion`, `Category`)
- **`.github/DISCUSSION_TEMPLATE/`** - Discussion form templates for 20+ product categories

## Key Workflows & Automation

### Discussion Lifecycle Management

**Dormant Discussion Handling** (daily at 01:25 UTC):
1. `comment-on-dormant-discussions.yml` - Labels discussions inactive after 60 days of no updates
2. `close-dormant-discussions.yml` - Closes discussions after 90 days (30 days post-labeling)
3. `remove-inactive-label.yml` - Removes "inactive" label if user responds

**Feedback Flow**:
- `add-feedback-comment.yml` - Auto-posts feedback acknowledgment when labeled
- `label-templated-discussions.yml` - Auto-labels based on discussion template selections (uses jsdom to parse HTML body)

### Incident Discussion Management

Special workflows for GitHub incident tracking:
- `open-incident-discussion.yml` - Creates incident discussion with labels
- `update-incident-discussion.yml` - Updates status as investigations progress
- `update-incident-discussion-as-resolved.yml` - Marks resolved with image banner
- `post-incident-summary.yml` - Posts post-mortem summary
- `close-incident-discussions.yml` - Archives old incidents

### Auto-Labeling Systems

- **Templated**: Parses HTML body (`<h3>Select Topic Area</h3>`) and applies labels
- **Product-specific**: `Accessibility-Labeler.yml`, `actions_labeller.yml`, `copilot_labeller.yml`
- **Governance**: `remove-governed-labels-on-create.yml` prevents users from setting protected labels

## Ruby Code Patterns

### GraphQL API Wrapper (`lib/github.rb`)

All API calls go through the `GitHub` class:
```ruby
GitHub.new.post(graphql: query)  # Queries with pagination
GitHub.new.mutate(graphql: mutation)  # Mutations
```

**Important**: Queries auto-paginate using `%ENDCURSOR%` placeholder and return flattened node arrays.

### Discussion Queries (`lib/discussions.rb`)

Uses **Struct-based models** (not classes):
```ruby
Discussion = Struct.new(:id, :url, :title, :labelled, :body, :created_at, :is_answered)
Category = Struct.new(:id, :name, :answerable, :discussions)
```

**Common patterns**:
- `.all()` - Finds unanswered discussions with "Question" label updated <60 days ago
- `.to_be_closed()` - Finds discussions inactive for 90+ days
- `.should_comment?()` - Checks if bot already commented (prevents spam)

**Category search strings** must escape quotes: `category:\\\"Projects and Issues\\\"`

### Script Conventions

Action scripts (e.g., `actions/add_feedback_comment`):
- Shebang: `#!/usr/bin/env ruby`
- Frozen string literals: `# frozen_string_literal: true`
- Read ENV vars: `ENV['GITHUB_TOKEN']`, `ENV['DISCUSSION_NUMBER']`
- Use `require_relative "../lib/github"` for shared code

## Development Workflows

### Running Actions Locally

```bash
# Set required env vars
$env:GITHUB_TOKEN = "ghp_..."
$env:DISCUSSION_NUMBER = "12345"
$env:NODE_ID = "D_kwDOE..."

# Run script directly
ruby .github/actions/add_feedback_comment
```

### Testing with Bundler

```bash
bundle install
bundle exec rubocop  # Lint with GitHub style rules
```

Linting config: `.rubocop.yml` inherits `rubocop-github` (config/default.yml + config/rails.yml)

### Adding New Workflows

1. **Create workflow** in `.github/workflows/` with proper permissions:
   ```yaml
   permissions:
     contents: read
     discussions: write
   ```

2. **Add Ruby script** in `.github/actions/` (make executable on Unix)

3. **Use shared libs**: `require_relative "../lib/github"` for GraphQL client

4. **Test query scope**: Verify category filters and date ranges in search strings

## Discussion Templates

Each template (`.github/DISCUSSION_TEMPLATE/*.yml`) defines:
- **labels**: Auto-applied labels (e.g., `[Copilot]`)
- **body**: Form fields (dropdown, textarea, markdown)
  - Use dropdowns for auto-labeling (parsed by `label-templated-discussions.yml`)

**Example structure**:
```yaml
labels: [Copilot]
body:
  - type: dropdown
    attributes:
      label: Select Topic Area
      options: ["Question", "Product Feedback", "Bug"]
```

## Common Gotchas

- **Rate limiting**: Scripts print rate limit info to stdout (`Rate limit: remaining - X`)
- **Pagination**: Use `%ENDCURSOR%` in queries; don't implement manual pagination
- **Bot detection**: Check `author.login == "github-actions"` to avoid processing bot comments
- **Label IDs**: Fetch label IDs via GraphQL before applying (stored as node IDs, not names)
- **HTML parsing**: Use jsdom (installed via npm in workflows) to parse `bodyHTML` from discussions

## External Dependencies

- **faraday** - HTTP client for GraphQL requests
- **active_support** - Date/time calculations (`.advance(days: -60)`)
- **rubocop-github** - Linting rules
- **jsdom** (npm) - HTML parsing in JavaScript actions

## Important Files to Reference

- `lib/discussions.rb` (530 lines) - All discussion query/mutation logic
- `workflows/label-templated-discussions.yml` - Shows HTML parsing pattern
- `actions/add_feedback_comment` - Canonical script structure
- `DISCUSSION_TEMPLATE/general.yml` - Template form example
