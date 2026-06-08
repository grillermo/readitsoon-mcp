# Unified Send Limits & Dedup Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Unify limit enforcement, usage counting, and dedup behavior across Kindle web, Kindle MCP, and Obsidian send flows so all three behave identically.

**Architecture:** Extract two reusable abstractions — `ArticleLimitable` concern (shared limit logic) and `RecentSendGuard` service (1-min dedup cache) — then apply them uniformly across Email and ObsidianPluginToken models and their three controllers/tool. Free accounts (no subscription period) get capped at 10/month by calendar month; paid accounts use their Stripe window. All sends emit ahoy analytics.

**Tech Stack:** Rails 8.1, RSpec, Rails cache, Ahoy analytics, Doorkeeper (OAuth).

---

## File Structure

**New files:**
- `app/models/concerns/article_limitable.rb` — Shared limit logic (concern)
- `app/services/recent_send_guard.rb` — Dedup service
- `spec/models/concerns/article_limitable_spec.rb` — Concern tests
- `spec/services/recent_send_guard_spec.rb` — Service tests

**Modified files:**
- `app/models/email.rb` — Include concern, remove old methods
- `app/models/obsidian_plugin_token.rb` — Include concern, remove duplicate methods
- `app/controllers/articles_controller.rb` — Use guard + new concern API
- `app/mcp/send_markdown_to_kindle_tool.rb` — Use guard + concern, add ahoy
- `app/controllers/obsidian_articles_controller.rb` — Use guard, already has concern-compatible code
- `spec/requests/articles_spec.rb` — Update limit check assertions
- `spec/requests/obsidian_articles_spec.rb` — Update for dedup
- `spec/mcp/send_markdown_to_kindle_tool_spec.rb` — Add dedup + ahoy + free-cap tests

---

## Task 1: Create ArticleLimitable concern with tests

**Files:**
- Create: `app/models/concerns/article_limitable.rb`
- Create: `spec/models/concerns/article_limitable_spec.rb`

- [ ] **Step 1: Write failing test for subscription_period_set?**

Create `spec/models/concerns/article_limitable_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe ArticleLimitable do
  let(:email) { Email.create!(email: "test@example.com") }
  let(:token) { ObsidianPluginToken.create!(token: "x" * 20) }

  describe "#subscription_period_set?" do
    context "without subscription period" do
      it "returns false when both dates are nil" do
        expect(email.subscription_period_set?).to be false
      end

      it "returns false when only start is set" do
        email.update!(current_subscription_period_start: Time.current)
        expect(email.subscription_period_set?).to be false
      end

      it "returns false when only end is set" do
        email.update!(current_subscription_period_end: Time.current)
        expect(email.subscription_period_set?).to be false
      end
    end

    context "with subscription period" do
      it "returns true when both dates are set" do
        email.update!(
          current_subscription_period_start: 1.month.ago,
          current_subscription_period_end: 1.month.from_now
        )
        expect(email.subscription_period_set?).to be true
      end
    end
  end
end
```

- [ ] **Step 2: Run test to verify it fails**

```bash
cd /Users/grillermo/c/readitsoon/.worktrees/mcp-oauth && \
  (set -a; . ./.env; set +a; RAILS_ENV=test bundle exec rspec spec/models/concerns/article_limitable_spec.rb::ArticleLimitable -v --no-color 2>&1 | head -20)
```

Expected: FAIL — `ArticleLimitable` uninitialized constant.

- [ ] **Step 3: Create the concern file**

Create `app/models/concerns/article_limitable.rb`:

```ruby
module ArticleLimitable
  extend ActiveSupport::Concern

  def subscription_period_set?
    current_subscription_period_start.present? && current_subscription_period_end.present?
  end

  def articles_this_period
    window = if subscription_period_set?
      current_subscription_period_start..current_subscription_period_end
    else
      Time.current.beginning_of_month..Time.current.end_of_month
    end
    articles.where(created_at: window).count
  end

  def over_limit?
    articles_this_period >= max_articles_per_month
  end
end
```

- [ ] **Step 4: Update Email to include the concern**

Edit `app/models/email.rb` — add `include ArticleLimitable` at the top of the class (after `devise` line):

```ruby
class Email < ApplicationRecord
  SIGN_IN_OTP_LENGTH = 6
  SIGN_IN_OTP_TTL = 1.hour

  devise :database_authenticatable, authentication_keys: [ :email ]
  include KindleEmailValidatable
  include ArticleLimitable  # <-- ADD THIS

  before_validation :ensure_token, on: :create
  has_many :articles, dependent: :destroy
  # ... rest of model
```

- [ ] **Step 5: Update ObsidianPluginToken to include the concern**

Edit `app/models/obsidian_plugin_token.rb` — add `include ArticleLimitable` at the top:

```ruby
class ObsidianPluginToken < ApplicationRecord
  include ArticleLimitable  # <-- ADD THIS

  before_validation :ensure_token, on: :create
  has_many :articles, dependent: :destroy
  # ... rest of model
```

- [ ] **Step 6: Run test to verify it passes**

```bash
cd /Users/grillermo/c/readitsoon/.worktrees/mcp-oauth && \
  (set -a; . ./.env; set +a; RAILS_ENV=test bundle exec rspec spec/models/concerns/article_limitable_spec.rb::ArticleLimitable -v --no-color 2>&1 | grep -E "passed|failed")
```

Expected: PASS (1 example).

- [ ] **Step 7: Add test for articles_this_period (non-subscriber, calendar month)**

Append to `spec/models/concerns/article_limitable_spec.rb` inside the describe block:

```ruby
  describe "#articles_this_period" do
    context "without subscription period (free account)" do
      before do
        email.update!(current_subscription_period_start: nil, current_subscription_period_end: nil)
        # Create articles: 2 this month, 1 last month
        email.articles.create!(url: "http://a.com", title: "A", markdown: "a", sent_status: :scheduled)
        email.articles.create!(url: "http://b.com", title: "B", markdown: "b", sent_status: :scheduled)
        email.articles.update_all(created_at: 35.days.ago)  # Move to last month
        email.articles.first.update!(created_at: Time.current)  # Keep one in this month
      end

      it "counts articles created in the current calendar month only" do
        expect(email.articles_this_period).to eq(1)
      end
    end

    context "with subscription period (paid account)" do
      before do
        period_start = 20.days.ago
        period_end = 10.days.from_now
        email.update!(
          current_subscription_period_start: period_start,
          current_subscription_period_end: period_end
        )
        # Create 3 articles: 2 in window, 1 before
        email.articles.create!(url: "http://a.com", title: "A", markdown: "a", sent_status: :scheduled, created_at: 30.days.ago)
        email.articles.create!(url: "http://b.com", title: "B", markdown: "b", sent_status: :scheduled, created_at: 15.days.ago)
        email.articles.create!(url: "http://c.com", title: "C", markdown: "c", sent_status: :scheduled, created_at: 5.days.ago)
      end

      it "counts articles created within the subscription window" do
        expect(email.articles_this_period).to eq(2)
      end
    end
  end
```

- [ ] **Step 8: Add test for over_limit?**

Append to the describe block:

```ruby
  describe "#over_limit?" do
    context "free account at exactly the limit" do
      before do
        email.update!(current_subscription_period_start: nil, current_subscription_period_end: nil, max_articles_per_month: 10)
        10.times { |i| email.articles.create!(url: "http://a#{i}.com", title: "A#{i}", markdown: "a", sent_status: :scheduled) }
      end

      it "returns true when articles_this_period >= max_articles_per_month" do
        expect(email.over_limit?).to be true
      end
    end

    context "free account below the limit" do
      before do
        email.update!(current_subscription_period_start: nil, current_subscription_period_end: nil, max_articles_per_month: 10)
        9.times { |i| email.articles.create!(url: "http://a#{i}.com", title: "A#{i}", markdown: "a", sent_status: :scheduled) }
      end

      it "returns false when articles_this_period < max_articles_per_month" do
        expect(email.over_limit?).to be false
      end
    end
  end
```

- [ ] **Step 9: Run all concern tests to verify they pass**

```bash
cd /Users/grillermo/c/readitsoon/.worktrees/mcp-oauth && \
  (set -a; . ./.env; set +a; RAILS_ENV=test bundle exec rspec spec/models/concerns/article_limitable_spec.rb -v --no-color 2>&1 | tail -5)
```

Expected: All examples pass.

- [ ] **Step 10: Commit**

```bash
cd /Users/grillermo/c/readitsoon/.worktrees/mcp-oauth && \
git add app/models/concerns/article_limitable.rb spec/models/concerns/article_limitable_spec.rb && \
git commit -m "feat: ArticleLimitable concern for shared limit logic

Extracted subscription_period_set?, articles_this_period, and over_limit?
into a reusable concern. Supports both paid (subscription window) and
free (calendar month) counting. Default max_articles_per_month=10.

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## Task 2: Create RecentSendGuard service with tests

**Files:**
- Create: `app/services/recent_send_guard.rb`
- Create: `spec/services/recent_send_guard_spec.rb`

- [ ] **Step 1: Write failing test for RecentSendGuard**

Create `spec/services/recent_send_guard_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe RecentSendGuard do
  let(:guard) { RecentSendGuard.new("test@example.com") }

  describe "#recently_sent?" do
    it "returns false for a new send (no cache)" do
      expect(guard.recently_sent?(markdown: "content", url: "http://a.com")).to be false
    end

    it "returns true for the same url within 1 minute" do
      guard.mark_sent!(markdown: "content", url: "http://a.com")
      expect(guard.recently_sent?(markdown: "content", url: "http://a.com")).to be true
    end

    it "returns true for the same markdown (url blank) within 1 minute" do
      guard.mark_sent!(markdown: "unique-md-content", url: "")
      expect(guard.recently_sent?(markdown: "unique-md-content", url: "")).to be true
    end

    it "returns false for a different url" do
      guard.mark_sent!(markdown: "content", url: "http://a.com")
      expect(guard.recently_sent?(markdown: "content", url: "http://b.com")).to be false
    end

    it "returns false for a different owner" do
      guard.mark_sent!(markdown: "content", url: "http://a.com")
      other_guard = RecentSendGuard.new("other@example.com")
      expect(other_guard.recently_sent?(markdown: "content", url: "http://a.com")).to be false
    end

    it "returns false after TTL expires" do
      guard.mark_sent!(markdown: "content", url: "http://a.com")
      travel 70.seconds
      expect(guard.recently_sent?(markdown: "content", url: "http://a.com")).to be false
    end
  end

  describe "#mark_sent!" do
    it "marks a url as sent for the owner" do
      guard.mark_sent!(markdown: "content", url: "http://a.com")
      expect(guard.recently_sent?(markdown: "content", url: "http://a.com")).to be true
    end

    it "uses md5 hash for markdown when url is blank" do
      md = "long markdown content that should be hashed"
      guard.mark_sent!(markdown: md, url: "")
      expect(guard.recently_sent?(markdown: md, url: "")).to be true
    end
  end
end
```

- [ ] **Step 2: Run test to verify it fails**

```bash
cd /Users/grillermo/c/readitsoon/.worktrees/mcp-oauth && \
  (set -a; . ./.env; set +a; RAILS_ENV=test bundle exec rspec spec/services/recent_send_guard_spec.rb -v --no-color 2>&1 | head -20)
```

Expected: FAIL — `RecentSendGuard` uninitialized constant.

- [ ] **Step 3: Create RecentSendGuard service**

Create `app/services/recent_send_guard.rb`:

```ruby
class RecentSendGuard
  TTL = 1.minute

  def initialize(owner_key)
    @owner_key = owner_key
  end

  def recently_sent?(markdown:, url:)
    Rails.cache.exist?(cache_key(markdown, url))
  end

  def mark_sent!(markdown:, url:)
    Rails.cache.write(cache_key(markdown, url), true, expires_in: TTL)
  end

  private

  def cache_key(markdown, url)
    base = if url.to_s.empty?
      "md:#{Digest::MD5.hexdigest(markdown.to_s)}"
    else
      "url:#{url}"
    end
    "recent_send:#{@owner_key}:#{base}"
  end
end
```

- [ ] **Step 4: Run test to verify it passes**

```bash
cd /Users/grillermo/c/readitsoon/.worktrees/mcp-oauth && \
  (set -a; . ./.env; set +a; RAILS_ENV=test bundle exec rspec spec/services/recent_send_guard_spec.rb -v --no-color 2>&1 | tail -5)
```

Expected: All examples pass.

- [ ] **Step 5: Commit**

```bash
cd /Users/grillermo/c/readitsoon/.worktrees/mcp-oauth && \
git add app/services/recent_send_guard.rb spec/services/recent_send_guard_spec.rb && \
git commit -m "feat: RecentSendGuard service for 1-minute dedup

Extracts dedup logic from ArticlesController into a reusable service.
Keys by owner + (url or md5(markdown)) with 1-minute TTL. Used by all
three send flows (Kindle web, MCP, Obsidian).

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## Task 3: Remove duplicate methods from models and update callers

**Files:**
- Modify: `app/models/email.rb`
- Modify: `app/models/obsidian_plugin_token.rb`
- Modify: `app/controllers/previews_controller.rb` (if articles_in_period called)
- Modify: `app/controllers/obsidian_previews_controller.rb` (if articles_in_period called)

- [ ] **Step 1: Remove old methods from Email model**

Edit `app/models/email.rb` — delete these methods:

```ruby
def subscription_period_set?
  current_subscription_period_start.present? && current_subscription_period_end.present?
end

def articles_in_period
  return 0 unless subscription_period_set?
  articles.delivered.where(sent_at: current_subscription_period_start..current_subscription_period_end).count
end
```

After removal, Email should only have `include ArticleLimitable` to provide these methods.

- [ ] **Step 2: Remove duplicate methods from ObsidianPluginToken**

Edit `app/models/obsidian_plugin_token.rb` — delete:

```ruby
def subscription_period_set?
  current_subscription_period_start.present? && current_subscription_period_end.present?
end

def articles_this_period
  return articles.where(created_at: Time.current.beginning_of_month..Time.current.end_of_month).count unless subscription_period_set?
  articles.where(created_at: current_subscription_period_start..current_subscription_period_end).count
end

def over_limit?
  articles_this_period >= max_articles_per_month
end
```

ObsidianPluginToken now inherits these via `include ArticleLimitable`.

- [ ] **Step 3: Audit and update articles_in_period callers**

Search for `articles_in_period` in the codebase:

```bash
cd /Users/grillermo/c/readitsoon/.worktrees/mcp-oauth && \
grep -rn "articles_in_period" app/ spec/ --include="*.rb" | grep -v ".swp"
```

For each caller **outside Email/ObsidianPluginToken models**, update to use `over_limit?` or `articles_this_period`:

- `ArticlesController#send_to_kindle` (line ~19): Change `if email_record.articles_in_period >= email_record.max_articles_per_month` → `if email_record.over_limit?`
- `SendMarkdownToKindleTool` (line ~20): Change `if email.articles_in_period >= email.max_articles_per_month` → `if email.over_limit?`
- `app/controllers/previews_controller.rb` (if `articles_in_period` appears): likely reads the count for UI. Change to `articles_this_period`.
- `app/controllers/obsidian_previews_controller.rb` (if appears): same — use `articles_this_period`.

Check views for any hardcoded calls; if any, update them too.

- [ ] **Step 4: Test that changes compile without errors**

```bash
cd /Users/grillermo/c/readitsoon/.worktrees/mcp-oauth && \
  (set -a; . ./.env; set +a; RAILS_ENV=test bundle exec rspec spec/models/email_spec.rb -v --no-color 2>&1 | grep -E "error|Error|FAILED|passed" | head -10)
```

Expected: No errors, Email model loads cleanly.

- [ ] **Step 5: Commit**

```bash
cd /Users/grillermo/c/readitsoon/.worktrees/mcp-oauth && \
git add app/models/email.rb app/models/obsidian_plugin_token.rb app/controllers/articles_controller.rb app/mcp/send_markdown_to_kindle_tool.rb app/controllers/previews_controller.rb app/controllers/obsidian_previews_controller.rb && \
git commit -m "refactor: remove duplicate limit methods, use concern

Email and ObsidianPluginToken now inherit subscription_period_set?,
articles_this_period, and over_limit? via ArticleLimitable concern.
Updated all callers to use over_limit? for cap checks or articles_this_period
for the raw count. Behavior unchanged: free accounts capped at 10/month by
calendar month; paid accounts use subscription window.

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## Task 4: Add dedup to ArticlesController (Kindle web)

**Files:**
- Modify: `app/controllers/articles_controller.rb`
- Modify: `spec/requests/articles_spec.rb`

- [ ] **Step 1: Write a test for dedup in ArticlesController**

Edit `spec/requests/articles_spec.rb` — add a test (find or create the `send_to_kindle` describe block):

```ruby
RSpec.describe "ArticlesController#send_to_kindle" do
  # ... existing tests ...

  describe "dedup protection" do
    it "rejects duplicate sends within 1 minute" do
      email_record = Email.create!(email: "dup@example.com")
      url = "http://example.com/article"

      # First send succeeds
      post "/articles/send_to_kindle", params: {
        email: "dup@example.com",
        url: url,
        title: "Test",
        markdown: "Content"
      }
      expect(response).to have_http_status(:ok)
      expect(ActiveJob::Base.queue_adapter.enqueued_jobs.count).to eq(1)

      # Second send within 1 min returns already-sent response
      post "/articles/send_to_kindle", params: {
        email: "dup@example.com",
        url: url,
        title: "Test",
        markdown: "Content"
      }
      expect(response).to have_http_status(:ok)
      body = JSON.parse(response.body)
      expect(body["success"]).to be true
      expect(body["message"]).to include("recently")
      # Verify no second job was enqueued
      expect(ActiveJob::Base.queue_adapter.enqueued_jobs.count).to eq(1)
    end
  end
end
```

- [ ] **Step 2: Run test to verify it fails**

```bash
cd /Users/grillermo/c/readitsoon/.worktrees/mcp-oauth && \
  (set -a; . ./.env; set +a; RAILS_ENV=test bundle exec rspec spec/requests/articles_spec.rb -k "dedup" -v --no-color 2>&1 | tail -20)
```

Expected: FAIL — no dedup implemented yet; second send creates a second job.

- [ ] **Step 3: Implement dedup in ArticlesController**

Edit `app/controllers/articles_controller.rb` in the `send_to_kindle` method. Replace the inline dedup code (if any) with RecentSendGuard:

```ruby
def send_to_kindle
  # ... existing validation code ...

  guard = RecentSendGuard.new(@email)
  if guard.recently_sent?(markdown: @markdown, url: @url)
    return render json: { success: true, message: "Article was sent recently. Please try again in a moment." }
  end

  email_record = Email.find_by(email: @email)
  return head :not_found unless email_record

  if email_record.over_limit?
    return render json: { error: I18n.t("articles.send.limit_reached") }, status: :too_many_requests
  end

  article = email_record.articles.create!(
    url: @url,
    title: @title,
    author: @author.presence || @email.split("@").first,
    markdown: @markdown,
    sent_status: :scheduled
  )

  guard.mark_sent!(markdown: @markdown, url: @url)
  DeliveryJob.perform_later(article.id, @email)
  ahoy.track "article_sent", { url: @url, recipient: @email, source: "web" }

  render json: { success: true, message: "Article sent to #{@email}." }
end
```

(Replace the old `articles_in_period` check with `over_limit?` if not already done in Task 3.)

- [ ] **Step 4: Update the existing ahoy.track call to add source dimension**

If the existing `ahoy.track` call has `{ url: @url, recipient: @email }`, update it to include `source: "web"`:

```ruby
ahoy.track "article_sent", { url: @url, recipient: @email, source: "web" }
```

- [ ] **Step 5: Run the dedup test to verify it passes**

```bash
cd /Users/grillermo/c/readitsoon/.worktrees/mcp-oauth && \
  (set -a; . ./.env; set +a; RAILS_ENV=test bundle exec rspec spec/requests/articles_spec.rb -k "dedup" -v --no-color 2>&1 | tail -10)
```

Expected: PASS.

- [ ] **Step 6: Run all ArticlesController tests**

```bash
cd /Users/grillermo/c/readitsoon/.worktrees/mcp-oauth && \
  (set -a; . ./.env; set +a; RAILS_ENV=test bundle exec rspec spec/requests/articles_spec.rb -v --no-color 2>&1 | tail -5)
```

Expected: All pass; no regressions from the limit check change.

- [ ] **Step 7: Commit**

```bash
cd /Users/grillermo/c/readitsoon/.worktrees/mcp-oauth && \
git add app/controllers/articles_controller.rb spec/requests/articles_spec.rb && \
git commit -m "feat: add dedup + source dimension to Kindle web send

Use RecentSendGuard for 1-minute dedup of identical sends. Add 'source: web'
to ahoy tracking for channel breakdown. Replace articles_in_period check
with over_limit?().

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## Task 5: Add dedup & ahoy to SendMarkdownToKindleTool (MCP)

**Files:**
- Modify: `app/mcp/send_markdown_to_kindle_tool.rb`
- Modify: `spec/mcp/send_markdown_to_kindle_tool_spec.rb`

- [ ] **Step 1: Write test for dedup in MCP tool**

Edit `spec/mcp/send_markdown_to_kindle_tool_spec.rb` — add a test (or extend the existing tool test):

```ruby
RSpec.describe SendMarkdownToKindleTool do
  # ... existing setup ...

  describe "dedup protection" do
    it "returns already-sent response for duplicate markdown within 1 minute" do
      email = Email.create!(email: "mcp@example.com")
      markdown = "Test article content"

      # First call succeeds
      response1 = described_class.call(
        markdown: markdown,
        filename: "test.md",
        author: "Test",
        server_context: { email_id: email.id }
      )
      expect(response1.content.first[:text]).to include("Sending")
      expect(ActiveJob::Base.queue_adapter.enqueued_jobs.count).to eq(1)

      # Second call within 1 min returns dedup response
      response2 = described_class.call(
        markdown: markdown,
        filename: "test.md",
        author: "Test",
        server_context: { email_id: email.id }
      )
      expect(response2.content.first[:text]).to include("recently")
      # No second job enqueued
      expect(ActiveJob::Base.queue_adapter.enqueued_jobs.count).to eq(1)
    end
  end

  describe "free-tier cap" do
    it "rejects send when free account exceeds 10/month" do
      email = Email.create!(email: "free@example.com", max_articles_per_month: 10)
      # Create 10 articles in current month
      10.times { |i| email.articles.create!(url: "http://a#{i}.com", title: "A#{i}", markdown: "m#{i}", sent_status: :scheduled) }

      response = described_class.call(
        markdown: "Should fail",
        filename: "test.md",
        server_context: { email_id: email.id }
      )
      expect(response.is_error).to be true
      expect(response.content.first[:text]).to include("Monthly sending limit reached")
    end
  end

  describe "ahoy tracking" do
    it "emits article_sent event with source=mcp" do
      email = Email.create!(email: "track@example.com")
      allow(Ahoy::Tracker).to receive(:new).and_return(double("tracker"))

      expect_any_instance_of(Ahoy::Tracker).to receive(:track).with(
        "article_sent",
        hash_including(source: "mcp", recipient: "track@example.com")
      )

      described_class.call(
        markdown: "Test",
        filename: "test.md",
        server_context: { email_id: email.id }
      )
    end
  end
end
```

- [ ] **Step 2: Run test to verify it fails**

```bash
cd /Users/grillermo/c/readitsoon/.worktrees/mcp-oauth && \
  (set -a; . ./.env; set +a; RAILS_ENV=test bundle exec rspec spec/mcp/send_markdown_to_kindle_tool_spec.rb -k "dedup or free-tier or ahoy" -v --no-color 2>&1 | tail -20)
```

Expected: FAIL — no dedup, no ahoy tracking, free cap not enforced in MCP tool.

- [ ] **Step 3: Implement dedup, free cap, and ahoy in the tool**

Edit `app/mcp/send_markdown_to_kindle_tool.rb`:

```ruby
class SendMarkdownToKindleTool < MCP::Tool
  tool_name "send_markdown_to_kindle"
  title "Send Markdown to Kindle"
  input_schema(
    properties: {
      markdown: { type: "string", description: "Markdown content" },
      filename: { type: "string", description: "Article filename/title" },
      author: { type: "string", description: "Article author (optional)" }
    },
    required: %w[markdown filename]
  )

  def self.call(markdown:, filename:, author: nil, server_context:)
    email = Email.find_by(id: server_context[:email_id])
    return error("Not authenticated.") unless email

    return error("Markdown is empty.") if markdown.to_s.strip.empty?

    # Check dedup
    guard = RecentSendGuard.new(email.email)
    if guard.recently_sent?(markdown: markdown, url: "")
      return MCP::Tool::Response.new([{ type: "text", text: "Article was sent recently. Please try again in a moment." }])
    end

    # Check free-tier cap
    if email.over_limit?
      return error("Monthly sending limit reached.")
    end

    article = email.articles.create!(
      url: "",
      title: title_from(filename),
      author: author.presence || email.email.split("@").first,
      markdown: markdown,
      sent_status: :scheduled
    )

    # Mark as sent in dedup cache
    guard.mark_sent!(markdown: markdown, url: "")

    # Enqueue delivery
    DeliveryJob.perform_later(article.id, email.email)

    # Track in ahoy (best-effort, never fail the send on tracking error)
    begin
      Ahoy::Tracker.new.track("article_sent", {
        recipient: email.email,
        source: "mcp",
        article_id: article.id
      })
    rescue StandardError => e
      Rails.logger.warn("Ahoy tracking failed for MCP send: #{e.message}")
    end

    MCP::Tool::Response.new([{ type: "text", text: "Sending '#{article.title}' to #{email.email}." }])
  end

  private

  def self.error(message)
    MCP::Tool::Response.new([{ type: "text", text: message }], is_error: true)
  end

  def self.title_from(filename)
    filename.sub(/\.\w+$/, "").tr("_-", " ").titleize
  end
end
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
cd /Users/grillermo/c/readitsoon/.worktrees/mcp-oauth && \
  (set -a; . ./.env; set +a; RAILS_ENV=test bundle exec rspec spec/mcp/send_markdown_to_kindle_tool_spec.rb -k "dedup or free-tier or ahoy" -v --no-color 2>&1 | tail -10)
```

Expected: All pass.

- [ ] **Step 5: Run full MCP suite to check for regressions**

```bash
cd /Users/grillermo/c/readitsoon/.worktrees/mcp-oauth && \
  (set -a; . ./.env; set +a; RAILS_ENV=test bundle exec rspec spec/mcp/ -v --no-color 2>&1 | tail -5)
```

Expected: All pass.

- [ ] **Step 6: Commit**

```bash
cd /Users/grillermo/c/readitsoon/.worktrees/mcp-oauth && \
git add app/mcp/send_markdown_to_kindle_tool.rb spec/mcp/send_markdown_to_kindle_tool_spec.rb && \
git commit -m "feat: add dedup, free cap, ahoy tracking to MCP tool

Use RecentSendGuard for 1-minute dedup. Enforce free-tier cap via
over_limit?(). Emit article_sent event with source=mcp. Tracking is
best-effort and never fails the send.

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## Task 6: Add dedup to ObsidianArticlesController

**Files:**
- Modify: `app/controllers/obsidian_articles_controller.rb`
- Modify: `spec/requests/obsidian_articles_spec.rb`

- [ ] **Step 1: Write test for dedup in Obsidian flow**

Edit `spec/requests/obsidian_articles_spec.rb` — add a test:

```ruby
RSpec.describe "ObsidianArticlesController#send_to_obsidian" do
  # ... existing setup ...

  describe "dedup protection" do
    it "rejects duplicate sends within 1 minute" do
      token = ObsidianPluginToken.create!(token: "x" * 20)
      url = "http://example.com/article"
      markdown = "Test content"

      # First send succeeds
      post "/obsidian/articles", params: {
        url: url,
        markdown_content: markdown,
        title: "Test",
        author: "Author",
        token: token.token,
        signature: sign_obsidian_request(url, markdown, "Test", "Author", token.token)
      }
      expect(response).to have_http_status(:ok)
      expect(token.articles.count).to eq(1)

      # Second send within 1 min is dedupped
      post "/obsidian/articles", params: {
        url: url,
        markdown_content: markdown,
        title: "Test",
        author: "Author",
        token: token.token,
        signature: sign_obsidian_request(url, markdown, "Test", "Author", token.token)
      }
      expect(response).to have_http_status(:ok)
      # No second article created
      expect(token.articles.count).to eq(1)
    end
  end
end
```

- [ ] **Step 2: Run test to verify it fails**

```bash
cd /Users/grillermo/c/readitsoon/.worktrees/mcp-oauth && \
  (set -a; . ./.env; set +a; RAILS_ENV=test bundle exec rspec spec/requests/obsidian_articles_spec.rb -k "dedup" -v --no-color 2>&1 | tail -20)
```

Expected: FAIL — dedup not implemented; second send creates a second article.

- [ ] **Step 3: Implement dedup in ObsidianArticlesController**

Edit `app/controllers/obsidian_articles_controller.rb` in the `send_to_obsidian` method:

```ruby
def send_to_obsidian
  return unless ensure_valid_request

  obsidian_token = ObsidianPluginToken.find_by(token: @token)
  return head :not_found unless obsidian_token

  # Check dedup
  guard = RecentSendGuard.new(@token)
  if guard.recently_sent?(markdown: @markdown, url: @url)
    return render json: { status: "ok", message: "Article was saved recently." }
  end

  # Check limit
  if obsidian_token.over_limit?
    return render json: { error: "limit_exceeded" }, status: :unprocessable_content
  end

  article = obsidian_token.articles.create!(
    url: @url,
    title: @title,
    author: @author,
    markdown: @markdown
  )

  # Mark as sent in dedup cache
  guard.mark_sent!(markdown: @markdown, url: @url)

  ahoy.track "obsidian_article_saved", { url: @url, article_id: article.id }

  render json: { status: "ok" }
end
```

- [ ] **Step 4: Run test to verify it passes**

```bash
cd /Users/grillermo/c/readitsoon/.worktrees/mcp-oauth && \
  (set -a; . ./.env; set +a; RAILS_ENV=test bundle exec rspec spec/requests/obsidian_articles_spec.rb -k "dedup" -v --no-color 2>&1 | tail -10)
```

Expected: PASS.

- [ ] **Step 5: Run all Obsidian tests**

```bash
cd /Users/grillermo/c/readitsoon/.worktrees/mcp-oauth && \
  (set -a; . ./.env; set +a; RAILS_ENV=test bundle exec rspec spec/requests/obsidian_articles_spec.rb -v --no-color 2>&1 | tail -5)
```

Expected: All pass.

- [ ] **Step 6: Commit**

```bash
cd /Users/grillermo/c/readitsoon/.worktrees/mcp-oauth && \
git add app/controllers/obsidian_articles_controller.rb spec/requests/obsidian_articles_spec.rb && \
git commit -m "feat: add dedup to Obsidian send flow

Use RecentSendGuard for 1-minute dedup. Now all three flows (Kindle web,
MCP, Obsidian) have identical dedup and limit behavior.

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## Task 7: Final integration test and regressions

**Files:**
- Modify: (verify no changes needed, only run tests)

- [ ] **Step 1: Run full Kindle/MCP/Obsidian test suites**

```bash
cd /Users/grillermo/c/readitsoon/.worktrees/mcp-oauth && \
  (set -a; . ./.env; set +a; RAILS_ENV=test bundle exec rspec \
    spec/requests/articles_spec.rb \
    spec/requests/obsidian_articles_spec.rb \
    spec/mcp/ \
    spec/models/concerns/article_limitable_spec.rb \
    spec/services/recent_send_guard_spec.rb \
    -v --no-color 2>&1 | tail -10)
```

Expected: All pass.

- [ ] **Step 2: Run full model and concern tests**

```bash
cd /Users/grillermo/c/readitsoon/.worktrees/mcp-oauth && \
  (set -a; . ./.env; set +a; RAILS_ENV=test bundle exec rspec spec/models/ -v --no-color 2>&1 | tail -5)
```

Expected: All pass, no regressions.

- [ ] **Step 3: Quick regression: verify limit behavior hasn't changed for paid accounts**

Write a quick ad-hoc test in a Ruby console to verify subscription-window logic still works:

```bash
cd /Users/grillermo/c/readitsoon/.worktrees/mcp-oauth && \
  (set -a; . ./.env; set +a; RAILS_ENV=test bundle exec rails runner '
    require "digest"
    email = Email.create!(email: "paid@example.com", max_articles_per_month: 5)
    start = 10.days.ago
    finish = 10.days.from_now
    email.update!(current_subscription_period_start: start, current_subscription_period_end: finish)
    
    # Create 5 articles within the subscription window
    5.times { |i| email.articles.create!(url: "http://a#{i}.com", title: "A#{i}", markdown: "m#{i}", sent_status: :scheduled) }
    
    puts "subscription_period_set? = #{email.subscription_period_set?}"
    puts "articles_this_period = #{email.articles_this_period}"
    puts "over_limit? = #{email.over_limit?}"
    
    raise "PAID ACCOUNT FAILED" unless email.subscription_period_set? && email.articles_this_period == 5 && email.over_limit?
    
    puts "PAID ACCOUNT OK"
  ' 2>&1 | tail -10)
```

Expected: "PAID ACCOUNT OK".

- [ ] **Step 4: Verify free-tier cap is enforced**

```bash
cd /Users/grillermo/c/readitsoon/.worktrees/mcp-oauth && \
  (set -a; . ./.env; set +a; RAILS_ENV=test bundle exec rails runner '
    email = Email.create!(email: "free@example.com", max_articles_per_month: 10)
    email.update!(current_subscription_period_start: nil, current_subscription_period_end: nil)
    
    # Create 10 articles in current month
    10.times { |i| email.articles.create!(url: "http://a#{i}.com", title: "A#{i}", markdown: "m#{i}", sent_status: :scheduled) }
    
    puts "subscription_period_set? = #{email.subscription_period_set?}"
    puts "articles_this_period = #{email.articles_this_period}"
    puts "over_limit? = #{email.over_limit?}"
    
    raise "FREE ACCOUNT FAILED" unless !email.subscription_period_set? && email.articles_this_period == 10 && email.over_limit?
    
    puts "FREE ACCOUNT OK"
  ' 2>&1 | tail -10)
```

Expected: "FREE ACCOUNT OK".

- [ ] **Step 5: Commit any final adjustments (if needed)**

```bash
cd /Users/grillermo/c/readitsoon/.worktrees/mcp-oauth && \
git status
```

Expected: Clean (no uncommitted changes). If there are any, commit them with an appropriate message.

---

## Verification Checklist

Before marking the feature complete, verify:

- [ ] **All three flows use `over_limit?`**: ArticlesController, SendMarkdownToKindleTool, ObsidianArticlesController.
- [ ] **Dedup guard applied to all three**: RecentSendGuard used consistently across all send paths.
- [ ] **Ahoy tracking with source dimension**: Kindle web and MCP both emit `article_sent` with `source: "web"` or `source: "mcp"`. Obsidian still uses `obsidian_article_saved`.
- [ ] **Free cap enforced**: Non-subscribers capped at `max_articles_per_month` per calendar month.
- [ ] **Subscription window respected**: Paid accounts use their Stripe period for counting.
- [ ] **All tests pass**: Full suite, no regressions.
- [ ] **No `articles_in_period` remains** in codebase except in comments/docs.
- [ ] **Log output clean**: No warnings or errors in test runs.

---

## Notes

- The change from `delivered` (old Email behavior) to `created_at` (new behavior) means a scheduled-but-not-delivered send counts against quota. This aligns with ObsidianPluginToken behavior and is intentional.
- Ahoy tracking outside a controller context (MCP tool) is best-effort and will never fail the send.
- The dedup cache uses Redis (or in-memory cache if configured) with a 1-minute TTL. No database write.
- All three flows now share identical semantics: same cap, same dedup, same analytics.
