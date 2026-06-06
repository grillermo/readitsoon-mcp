# Instructions: unify limit / usage / dedup behavior across all three send flows

**Goal:** the three send paths must behave identically on three axes — **analytics (ahoy) tracking**, **`recently_sent?` dedup**, and **a real free-tier cap for non-subscribers via a shared `over_limit?`** that also counts usage for subscription purposes.

**Where:** readitsoon, branch `mcp-oauth`, worktree `/Users/grillermo/c/readitsoon/.worktrees/mcp-oauth`.
Run anything with: `cd <worktree> && (set -a; . ./.env; set +a; RAILS_ENV=test bundle exec rspec <path> --no-color)`.

## The three flows

1. **Kindle (web/bookmarklet):** `ArticlesController#send_to_kindle` — owner `Email`, enqueues `DeliveryJob`.
2. **Kindle (MCP):** `SendMarkdownToKindleTool` (`app/mcp/`) — owner `Email`, enqueues `DeliveryJob`.
3. **Obsidian:** `ObsidianArticlesController#send_to_obsidian` — owner `ObsidianPluginToken`, no delivery (saved for pull).

## Current state (what's inconsistent)

| Axis | Kindle web | MCP | Obsidian |
|---|---|---|---|
| ahoy track | ✅ `article_sent` | ❌ none | ✅ `obsidian_article_saved` |
| `recently_sent?` dedup | ✅ (1-min cache) | ❌ | ❌ |
| free cap for non-subscribers | ❌ `articles_in_period` returns 0 → uncapped | ❌ same | ✅ `over_limit?` monthly fallback |

`Email#articles_in_period` returns `0` unless `subscription_period_set?`, so free Kindle accounts are uncapped. `ObsidianPluginToken#articles_this_period` falls back to a **current-calendar-month** count for non-subscribers — that is the behavior we standardize on.

## Target behavior (all three identical)

1. **Cap:** before creating the article, reject if `owner.over_limit?`.
   - `over_limit?` ⇔ `articles_this_period >= max_articles_per_month`.
   - `articles_this_period` = count of the owner's articles by **`created_at`** within the **subscription window if `subscription_period_set?`, else the current calendar month**. This is a real cap for non-subscribers (default `max_articles_per_month = 10`).
2. **Dedup:** reject duplicate sends within **1 minute** keyed by owner + (url, or md5(markdown) when url blank), returning the existing "already sent" success shape per flow.
3. **Analytics:** every successful send emits an ahoy event (see "Event names" below).

## Shared abstractions to introduce (DRY)

### A. `ArticleLimitable` concern — `app/models/concerns/article_limitable.rb`
Include in **both** `Email` and `ObsidianPluginToken`. Provides:
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
- Remove the now-duplicated `subscription_period_set?` / `articles_this_period` / `over_limit?` from `ObsidianPluginToken` (it keeps behaving the same — already monthly-fallback + created_at).
- `Email`: remove `subscription_period_set?` and `articles_in_period`; include the concern. **Audit every `articles_in_period` caller** (`ArticlesController#send_to_kindle`, the MCP tool, `previews_controller`, `obsidian_previews_controller`, any view) and switch limit checks to `over_limit?`. If any caller needs the raw number, use `articles_this_period`.
  - NOTE: this changes Kindle counting from `delivered`-in-window to `created_at`-in-period. Intended: counts at send time, closes the free bypass. Confirm no view relies on the old delivered-only semantics; update accordingly.

### B. `RecentSendGuard` — `app/services/recent_send_guard.rb`
Extract `ArticlesController`'s `recently_sent?` / `calculate_sent_cache_key` into a reusable PORO:
```ruby
class RecentSendGuard
  TTL = 1.minute
  def initialize(owner_key) = @owner_key = owner_key   # e.g. email address or obsidian token
  def recently_sent?(markdown:, url:) = Rails.cache.exist?(key(markdown, url))
  def mark_sent!(markdown:, url:)      = Rails.cache.write(key(markdown, url), true, expires_in: TTL)
  private
  def key(markdown, url)
    base = url.to_s.empty? ? "md:#{Digest::MD5.hexdigest(markdown.to_s)}" : "url:#{url}"
    "recent_send:#{@owner_key}:#{base}"
  end
end
```
All three flows: `return <already-sent shape> if guard.recently_sent?(...)`, then `guard.mark_sent!(...)` after a successful enqueue/save. `ArticlesController` drops its private dedup methods and uses this.

### C. Ahoy from the MCP tool (non-controller context)
`ahoy` controller helper isn't available in `SendMarkdownToKindleTool`. Use a tracker directly:
```ruby
Ahoy::Tracker.new.track("article_sent", { recipient: email.email, source: "mcp", article_id: article.id })
```
Keep it best-effort (rescue/log on failure; never fail the send because analytics failed).

## Event names

- Kindle web + MCP: **`article_sent`** (add a `source:` dimension — `"web"` / `"mcp"` — to both for channel breakdown; recipient stays).
- Obsidian: keep **`obsidian_article_saved`** (distinct channel/event, unchanged).

"Same behavior" = each flow tracks its successful send; it does **not** mean collapsing the obsidian event into `article_sent`.

## Per-flow changes

**`Email`** (`app/models/email.rb`): include `ArticleLimitable`; delete its local `subscription_period_set?` + `articles_in_period`.
**`ObsidianPluginToken`**: include `ArticleLimitable`; delete its local copies.

**`ArticlesController#send_to_kindle`:**
- Replace `if email_record.articles_in_period >= email_record.max_articles_per_month` with `if email_record.over_limit?` (keep the existing `:too_many_requests` + `articles.send.limit_reached` response).
- Replace inline `recently_sent?`/cache-key/`Rails.cache.write` with `RecentSendGuard.new(@email)`.
- Keep `ahoy.track "article_sent", { url: @url, recipient: @email, source: "web" }`.

**`SendMarkdownToKindleTool`:**
- Replace the manual `articles_in_period >= max_articles_per_month` with `return error("Monthly sending limit reached.") if email.over_limit?`.
- Add `RecentSendGuard.new(email.email)`: if `recently_sent?(markdown:, url: "")` return a success-shaped `MCP::Tool::Response` ("Already sent recently."); else `mark_sent!` after enqueue.
- Add the `Ahoy::Tracker` call from section C after enqueue.

**`ObsidianArticlesController#send_to_obsidian`:**
- Add `RecentSendGuard.new(@token)` dedup (return `{ status: "ok" }` / an "already_saved" shape on hit) before the `over_limit?` check; `mark_sent!` after create.
- `over_limit?` and `ahoy.track "obsidian_article_saved"` already present — unchanged.

## Tests (RSpec, all must pass)

- **`spec/models/concerns/article_limitable_spec.rb`** (or via Email + ObsidianPluginToken specs): non-subscriber capped at `max_articles_per_month` per calendar month; subscriber capped within the Stripe window; `over_limit?` boundary at exactly the cap.
- **`spec/services/recent_send_guard_spec.rb`**: same owner+url within 1 min → `recently_sent?` true; different url/owner → false; key uses md5 when url blank.
- **Kindle web** request spec: free account over the monthly cap → `429`; duplicate within 1 min → already-sent response, no second `DeliveryJob`; `article_sent` tracked.
- **MCP** tool spec (extend `spec/mcp/send_markdown_to_kindle_tool_spec.rb`): non-subscriber over monthly cap → error response; duplicate within 1 min → success-shaped "already sent", single `DeliveryJob`; ahoy event emitted (stub `Ahoy::Tracker`).
- **Obsidian** request spec: duplicate within 1 min → no second article; `over_limit?` still enforced.
- Update any existing specs that asserted the old free-tier "uncapped" Kindle behavior or the `articles_in_period` name.

## Risks / notes

- **Behavior change:** free Kindle accounts become capped at 10/month (intended — closes the bypass). Verify no marketing/UX copy promises unlimited free sends; update `max_articles` display in `previews_controller` if it reads the removed method.
- Counting switches to `created_at` (counts at send attempt) rather than `delivered` `sent_at`. A scheduled-but-failed send now consumes quota; acceptable and matches obsidian. If product wants delivered-only counting, that's a separate decision applied uniformly in the concern.
- Ahoy outside a request has no visit/visitor context; keep tracking best-effort and never let it raise into the send path.
- After changes: run the full `spec/requests/mcp spec/mcp` suite plus `spec/requests/articles*`, `spec/requests/obsidian*`, and the new model/service specs; then the full `bundle exec rspec` for regressions.
