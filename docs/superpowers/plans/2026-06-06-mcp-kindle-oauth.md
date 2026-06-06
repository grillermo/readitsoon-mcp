# MCP Kindle Send + OAuth 2.1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship a remote HTTP MCP server with a `send_markdown_to_kindle` tool, backed by an OAuth 2.1 Authorization Server added to readitsoon, reusing its markdown→EPUB→Kindle pipeline.

**Architecture:** Two repos. **readitsoon** (Rails) becomes the OAuth 2.1 Authorization Server (Doorkeeper) plus a token-guarded `Mcp::SendController`; all new Rails code lives under an `Mcp::` namespace and reuses existing EPUB/OTP/delivery services. **readitsoon-mcp** (TypeScript) is a Streamable-HTTP MCP server acting as OAuth Resource Server: it serves RFC 9728 protected-resource metadata, guards its `/mcp` endpoint with bearer tokens validated via readitsoon's RFC 7662 introspection endpoint, and exposes the send tool which POSTs to `/mcp/send`.

**Tech Stack:** Ruby on Rails, Doorkeeper, RSpec, Mailgun, Pandoc (existing). TypeScript, `@modelcontextprotocol/sdk`, Express, Vitest, `zod`.

**Repo / branching:**
- **readitsoon-mcp** (this repo): commit to `main`, atomic commits. This is Phase B.
- **readitsoon** (Rails): work in a git worktree under `../readitsoon/.worktrees` on branch `mcp-oauth`. This is Phase A. Create the worktree at execution start via `superpowers:using-git-worktrees`.

**Spec:** `docs/superpowers/specs/2026-06-06-mcp-kindle-oauth-design.md`

**Reuse map (readitsoon, do not reimplement):**
- `EpubCreator.convert(markdown, title, author, url)` → `[epub_path, dir]`
- `DeliveryService.call(epub_path, dir, url, title, recipient)`
- `Email#issue_sign_in_otp!` → otp string; `Email#valid_sign_in_otp?(otp)`; `Email#consume_sign_in_otp!`; `Email#token`
- `OtpSignInDeliveryService.call(email_record, otp)` (sends OTP as epub)
- `SignInIpGuard.new(request.remote_ip)` → `#banned?`, `#register_failure!`, `#reset_failures!`

---

## PHASE A — readitsoon (Rails Authorization Server)

> Run all Phase A tasks inside the `../readitsoon/.worktrees/mcp-oauth` worktree on branch `mcp-oauth`. Commits here are atomic, on that branch.

### Task A1: Install Doorkeeper

**Files:**
- Modify: `Gemfile`
- Create: `config/initializers/doorkeeper.rb` (generated)
- Create: `db/migrate/*_create_doorkeeper_tables.rb` (generated)
- Modify: `config/routes.rb`

- [ ] **Step 1: Add the gem**

Add to `Gemfile` near the other auth gems (`gem "devise"`):

```ruby
gem "doorkeeper", "~> 5.8"
```

- [ ] **Step 2: Install**

Run: `bundle install`
Expected: resolves, installs doorkeeper 5.8.x.

- [ ] **Step 3: Generate Doorkeeper install**

Run: `bin/rails generate doorkeeper:install`
Expected: creates `config/initializers/doorkeeper.rb` and adds `use_doorkeeper` to `config/routes.rb`.

- [ ] **Step 4: Generate migration**

Run: `bin/rails generate doorkeeper:migration`
Expected: creates `db/migrate/*_create_doorkeeper_tables.rb`.

- [ ] **Step 5: Make the migration support public clients + PKCE**

Open the generated `*_create_doorkeeper_tables.rb`. Ensure these columns are present and uncommented:
- In `oauth_applications`: `t.boolean :confidential, null: false, default: true` — change default to `false` (our clients are public). Keep `t.string :secret`.
- In `oauth_access_grants`: uncomment `t.string :code_challenge` and `t.string :code_challenge_method` (PKCE columns).

- [ ] **Step 6: Migrate**

Run: `bin/rails db:migrate`
Expected: tables `oauth_applications`, `oauth_access_grants`, `oauth_access_tokens` created.

- [ ] **Step 7: Commit**

```bash
git add Gemfile Gemfile.lock config/initializers/doorkeeper.rb config/routes.rb db/migrate db/schema.rb
git commit -m "feat(mcp): install doorkeeper with PKCE + public client support"
```

---

### Task A2: Configure Doorkeeper for the MCP OAuth flow

**Files:**
- Modify: `config/initializers/doorkeeper.rb`
- Test: `spec/requests/mcp/oauth_metadata_spec.rb`

- [ ] **Step 1: Write the failing test (AS metadata is reachable)**

Create `spec/requests/mcp/oauth_metadata_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe "OAuth Authorization Server metadata", type: :request do
  it "serves RFC 8414 metadata advertising introspection + registration" do
    get "/.well-known/oauth-authorization-server"

    expect(response).to have_http_status(:ok)
    body = JSON.parse(response.body)
    expect(body["issuer"]).to be_present
    expect(body["authorization_endpoint"]).to include("/oauth/authorize")
    expect(body["token_endpoint"]).to include("/oauth/token")
    expect(body["introspection_endpoint"]).to include("/oauth/introspect")
    expect(body["registration_endpoint"]).to include("/oauth/register")
    expect(body["code_challenge_methods_supported"]).to include("S256")
  end
end
```

- [ ] **Step 2: Run it, verify it fails**

Run: `bundle exec rspec spec/requests/mcp/oauth_metadata_spec.rb`
Expected: FAIL (route/endpoint missing).

- [ ] **Step 3: Configure the initializer**

Replace the body of `config/initializers/doorkeeper.rb` `Doorkeeper.configure do ... end` so it contains (keep generator comments you want, but these settings are required):

```ruby
Doorkeeper.configure do
  orm :active_record

  # Custom login: hand control to our OTP auth-session flow.
  # `current_mcp_resource_owner` is set by Mcp::AuthSessionsController after OTP verification.
  resource_owner_authenticator do
    if session[:mcp_authenticated_email_id]
      Email.find_by(id: session[:mcp_authenticated_email_id])
    else
      redirect_to(mcp_auth_session_path(return_to: request.fullpath))
      nil
    end
  end

  # Public native/desktop clients: no client secret, PKCE required.
  force_pkce
  grant_flows %w[authorization_code]

  # Allow loopback + custom-scheme redirect URIs used by MCP clients.
  # (Doorkeeper validates redirect_uri against the registered application.)
  allow_grant_flow_for_client { |_grant_flow, _client| true }

  # Tokens
  access_token_expires_in 2.hours
  use_refresh_token

  # Introspection: allow a resource server presenting a valid access token
  # (the token it is introspecting) to introspect. We additionally gate by
  # a shared secret header in Mcp::IntrospectionController is NOT used here;
  # Doorkeeper's default introspection requires client/token auth.
  # Permit introspection when the caller supplies any valid access token:
  allow_token_introspection do |token, authorized_client, authorized_token|
    authorized_token.present? || authorized_client.present?
  end

  # We do not use a separate admin UI; skip the protected resources area.
  base_controller "ActionController::Base"
end
```

- [ ] **Step 4: Mount metadata + introspection routes**

In `config/routes.rb`, the `use_doorkeeper` block must expose authorize/token/introspect, and we add the AS metadata route. Replace the generated `use_doorkeeper` line with:

```ruby
use_doorkeeper do
  controllers tokens: "mcp/tokens"   # see Task A6 (binds token to Email)
end

get "/.well-known/oauth-authorization-server", to: "mcp/metadata#authorization_server"
```

- [ ] **Step 5: Create the metadata controller**

Create `app/controllers/mcp/metadata_controller.rb`:

```ruby
module Mcp
  class MetadataController < ActionController::API
    def authorization_server
      issuer = request.base_url
      render json: {
        issuer: issuer,
        authorization_endpoint: "#{issuer}/oauth/authorize",
        token_endpoint: "#{issuer}/oauth/token",
        introspection_endpoint: "#{issuer}/oauth/introspect",
        registration_endpoint: "#{issuer}/oauth/register",
        response_types_supported: ["code"],
        grant_types_supported: %w[authorization_code refresh_token],
        code_challenge_methods_supported: ["S256"],
        token_endpoint_auth_methods_supported: ["none"]
      }
    end
  end
end
```

- [ ] **Step 6: Run the test, verify pass**

Run: `bundle exec rspec spec/requests/mcp/oauth_metadata_spec.rb`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add config/initializers/doorkeeper.rb config/routes.rb app/controllers/mcp/metadata_controller.rb spec/requests/mcp/oauth_metadata_spec.rb
git commit -m "feat(mcp): configure doorkeeper AS + serve authorization-server metadata"
```

---

### Task A3: Dynamic Client Registration (RFC 7591)

**Files:**
- Create: `app/controllers/mcp/registrations_controller.rb`
- Modify: `config/routes.rb`
- Test: `spec/requests/mcp/registrations_spec.rb`

- [ ] **Step 1: Write the failing test**

Create `spec/requests/mcp/registrations_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe "Dynamic Client Registration", type: :request do
  it "registers a public client and returns client_id (no secret)" do
    post "/oauth/register", params: {
      client_name: "Claude Code",
      redirect_uris: ["http://127.0.0.1:33418/callback"],
      token_endpoint_auth_method: "none",
      grant_types: ["authorization_code", "refresh_token"],
      response_types: ["code"]
    }.to_json, headers: { "Content-Type" => "application/json" }

    expect(response).to have_http_status(:created)
    body = JSON.parse(response.body)
    expect(body["client_id"]).to be_present
    expect(body["client_secret"]).to be_nil
    expect(body["redirect_uris"]).to eq(["http://127.0.0.1:33418/callback"])
    expect(Doorkeeper::Application.find_by(uid: body["client_id"])).to be_present
  end

  it "rejects registration with no redirect_uris" do
    post "/oauth/register", params: { client_name: "x" }.to_json,
         headers: { "Content-Type" => "application/json" }
    expect(response).to have_http_status(:bad_request)
    expect(JSON.parse(response.body)["error"]).to eq("invalid_redirect_uri")
  end
end
```

- [ ] **Step 2: Run it, verify it fails**

Run: `bundle exec rspec spec/requests/mcp/registrations_spec.rb`
Expected: FAIL (no `/oauth/register` route).

- [ ] **Step 3: Add the route**

In `config/routes.rb`, below the metadata route:

```ruby
post "/oauth/register", to: "mcp/registrations#create"
```

- [ ] **Step 4: Implement the controller**

Create `app/controllers/mcp/registrations_controller.rb`:

```ruby
module Mcp
  class RegistrationsController < ActionController::API
    def create
      redirect_uris = Array(params[:redirect_uris]).map(&:to_s).reject(&:blank?)
      if redirect_uris.empty?
        return render json: { error: "invalid_redirect_uri",
                              error_description: "redirect_uris is required" },
                      status: :bad_request
      end

      app = Doorkeeper::Application.new(
        name: params[:client_name].presence || "MCP Client",
        redirect_uri: redirect_uris.join("\n"),
        confidential: false,
        scopes: ""
      )

      unless app.save
        return render json: { error: "invalid_client_metadata",
                              error_description: app.errors.full_messages.join(", ") },
                      status: :bad_request
      end

      render json: {
        client_id: app.uid,
        client_id_issued_at: app.created_at.to_i,
        redirect_uris: redirect_uris,
        token_endpoint_auth_method: "none",
        grant_types: %w[authorization_code refresh_token],
        response_types: ["code"],
        client_name: app.name
      }, status: :created
    end
  end
end
```

- [ ] **Step 5: Run the test, verify pass**

Run: `bundle exec rspec spec/requests/mcp/registrations_spec.rb`
Expected: PASS (both examples).

- [ ] **Step 6: Commit**

```bash
git add app/controllers/mcp/registrations_controller.rb config/routes.rb spec/requests/mcp/registrations_spec.rb
git commit -m "feat(mcp): add RFC 7591 dynamic client registration endpoint"
```

---

### Task A4: OTP auth-session — email entry + send OTP

**Files:**
- Create: `app/controllers/mcp/auth_sessions_controller.rb`
- Create: `app/views/mcp/auth_sessions/new.html.erb`
- Create: `app/views/mcp/auth_sessions/show.html.erb`
- Modify: `config/routes.rb`
- Test: `spec/requests/mcp/auth_sessions_spec.rb`

- [ ] **Step 1: Write the failing test (email entry issues OTP)**

Create `spec/requests/mcp/auth_sessions_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe "MCP auth session", type: :request do
  it "shows the email entry form" do
    get "/mcp/auth_session", params: { return_to: "/oauth/authorize?x=1" }
    expect(response).to have_http_status(:ok)
    expect(response.body).to include("kindle")
  end

  it "issues an OTP epub and advances to the verify step" do
    expect(OtpSignInDeliveryService).to receive(:call).once

    post "/mcp/auth_session", params: { email: "reader@kindle.com", return_to: "/oauth/authorize?x=1" }

    expect(response).to redirect_to(mcp_auth_session_verify_path)
    email = Email.find_by(email: "reader@kindle.com")
    expect(email).to be_present
    expect(email.sign_in_otp_digest).to be_present
    expect(session[:mcp_otp_email_id]).to eq(email.id)
  end
end
```

- [ ] **Step 2: Run it, verify it fails**

Run: `bundle exec rspec spec/requests/mcp/auth_sessions_spec.rb`
Expected: FAIL (routes/controller missing).

- [ ] **Step 3: Add routes**

In `config/routes.rb`:

```ruby
get  "/mcp/auth_session",        to: "mcp/auth_sessions#new",    as: :mcp_auth_session
post "/mcp/auth_session",        to: "mcp/auth_sessions#create"
get  "/mcp/auth_session/verify", to: "mcp/auth_sessions#verify", as: :mcp_auth_session_verify
post "/mcp/auth_session/verify", to: "mcp/auth_sessions#confirm"
post "/mcp/auth_session/resend", to: "mcp/auth_sessions#resend", as: :mcp_auth_session_resend
```

- [ ] **Step 4: Implement `new` + `create` (and stubs for the rest, filled in A5)**

Create `app/controllers/mcp/auth_sessions_controller.rb`:

```ruby
module Mcp
  class AuthSessionsController < ApplicationController
    skip_before_action :track_ahoy_visit, raise: false
    skip_around_action :set_ahoy_request_store, raise: false
    skip_forgery_protection

    layout "application"

    RESEND_INTERVAL = 30.seconds

    def new
      session[:mcp_return_to] = params[:return_to] if params[:return_to].present?
      @email = ""
    end

    def create
      return render_ip_banned if ip_guard.banned?

      email_record = Email.find_or_initialize_by(email: email_param)
      unless email_record.save
        flash.now[:alert] = email_record.errors.full_messages.to_sentence
        @email = email_param
        return render :new, status: :unprocessable_content
      end

      issue_and_deliver_otp(email_record)
      session[:mcp_otp_email_id] = email_record.id
      session[:mcp_otp_sent_at]  = Time.current.to_i
      redirect_to mcp_auth_session_verify_path
    end

    # verify / confirm / resend implemented in Task A5

    private

    def issue_and_deliver_otp(email_record)
      otp = email_record.issue_sign_in_otp!
      OtpSignInDeliveryService.call(email_record, otp)
    end

    def email_param
      params[:email].to_s.strip
    end

    def ip_guard
      @ip_guard ||= SignInIpGuard.new(request.remote_ip)
    end

    def render_ip_banned
      render plain: "Too many attempts. Try later.", status: :too_many_requests
    end
  end
end
```

- [ ] **Step 5: Create the email-entry view**

Create `app/views/mcp/auth_sessions/new.html.erb`:

```erb
<main class="mcp-auth">
  <h1>Connect Read It Soon to Claude</h1>
  <p>Enter your Kindle email address. We'll send a one-time code as a document to your Kindle.</p>
  <p><strong>First:</strong> in your Amazon account, add our sender address to your
     approved senders list, or the code won't arrive on your Kindle.</p>

  <% if flash[:alert] %><p class="error"><%= flash[:alert] %></p><% end %>

  <%= form_with url: mcp_auth_session_path, method: :post, local: true do |f| %>
    <%= f.email_field :email, value: @email, placeholder: "you@kindle.com", required: true %>
    <%= f.submit "Send code" %>
  <% end %>
</main>
```

- [ ] **Step 6: Create a placeholder verify view (replaced in A5) so `redirect_to verify` renders**

Create `app/views/mcp/auth_sessions/show.html.erb` with a single line (kept minimal; A5 replaces with the real verify view named `verify.html.erb`):

```erb
<p>Enter the code we sent.</p>
```

- [ ] **Step 7: Run the test, verify pass**

Run: `bundle exec rspec spec/requests/mcp/auth_sessions_spec.rb`
Expected: PASS (first two examples).

- [ ] **Step 8: Commit**

```bash
git add app/controllers/mcp/auth_sessions_controller.rb app/views/mcp/auth_sessions config/routes.rb spec/requests/mcp/auth_sessions_spec.rb
git commit -m "feat(mcp): OTP auth-session email entry sends OTP epub"
```

---

### Task A5: OTP verify, confirm, and resend (30s gate)

**Files:**
- Modify: `app/controllers/mcp/auth_sessions_controller.rb`
- Create: `app/views/mcp/auth_sessions/verify.html.erb`
- Test: `spec/requests/mcp/auth_sessions_verify_spec.rb`

- [ ] **Step 1: Write the failing tests**

Create `spec/requests/mcp/auth_sessions_verify_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe "MCP auth session verify", type: :request do
  def start_session_with_otp(email_addr = "reader@kindle.com")
    allow(OtpSignInDeliveryService).to receive(:call)
    post "/mcp/auth_session", params: { email: email_addr, return_to: "/oauth/authorize?x=1" }
    Email.find_by(email: email_addr)
  end

  it "rejects an incorrect OTP" do
    start_session_with_otp
    post "/mcp/auth_session/verify", params: { otp: "wrong" }
    expect(response).to have_http_status(:unprocessable_content)
    expect(session[:mcp_authenticated_email_id]).to be_nil
  end

  it "accepts a correct OTP, marks the session authenticated, and returns to the OAuth flow" do
    email = start_session_with_otp
    otp = email.issue_sign_in_otp!  # known value
    allow_any_instance_of(Email).to receive(:valid_sign_in_otp?).and_return(true)

    post "/mcp/auth_session/verify", params: { otp: otp }

    expect(response).to redirect_to("/oauth/authorize?x=1")
    expect(session[:mcp_authenticated_email_id]).to eq(email.id)
  end

  it "rejects resend before 30s" do
    start_session_with_otp
    post "/mcp/auth_session/resend"
    expect(response).to have_http_status(:too_many_requests)
  end

  it "allows resend after 30s and re-sends the OTP epub" do
    email = start_session_with_otp
    # simulate the last send being >30s ago
    allow_any_instance_of(ActionDispatch::Request).to receive(:session)
      .and_wrap_original { |m| s = m.call; s[:mcp_otp_sent_at] = 31.seconds.ago.to_i; s }
    expect(OtpSignInDeliveryService).to receive(:call).once
    post "/mcp/auth_session/resend"
    expect(response).to redirect_to(mcp_auth_session_verify_path)
  end
end
```

- [ ] **Step 2: Run it, verify it fails**

Run: `bundle exec rspec spec/requests/mcp/auth_sessions_verify_spec.rb`
Expected: FAIL (`verify`/`confirm`/`resend` actions not implemented).

- [ ] **Step 3: Implement verify/confirm/resend**

In `app/controllers/mcp/auth_sessions_controller.rb`, replace the `# verify / confirm / resend implemented in Task A5` comment with:

```ruby
    def verify
      return redirect_to(mcp_auth_session_path) unless otp_email
      @resend_available_in = resend_seconds_remaining
    end

    def confirm
      return render_ip_banned if ip_guard.banned?
      email_record = otp_email
      return redirect_to(mcp_auth_session_path) unless email_record

      if email_record.valid_sign_in_otp?(params[:otp].to_s.strip)
        email_record.consume_sign_in_otp!
        ip_guard.reset_failures!
        session[:mcp_authenticated_email_id] = email_record.id
        session.delete(:mcp_otp_email_id)
        redirect_to(session.delete(:mcp_return_to) || "/")
      else
        ip_guard.register_failure!
        return render_ip_banned if ip_guard.banned?
        flash.now[:alert] = "That code didn't work. Try again."
        @resend_available_in = resend_seconds_remaining
        render :verify, status: :unprocessable_content
      end
    end

    def resend
      email_record = otp_email
      return redirect_to(mcp_auth_session_path) unless email_record
      if resend_seconds_remaining.positive?
        return render plain: "Wait before resending.", status: :too_many_requests
      end
      issue_and_deliver_otp(email_record)
      session[:mcp_otp_sent_at] = Time.current.to_i
      redirect_to mcp_auth_session_verify_path
    end
```

And add these private helpers (below `issue_and_deliver_otp`):

```ruby
    def otp_email
      id = session[:mcp_otp_email_id]
      id && Email.find_by(id: id)
    end

    def resend_seconds_remaining
      sent_at = session[:mcp_otp_sent_at].to_i
      remaining = RESEND_INTERVAL.to_i - (Time.current.to_i - sent_at)
      remaining.positive? ? remaining : 0
    end
```

- [ ] **Step 4: Create the verify view with the resend timer**

Create `app/views/mcp/auth_sessions/verify.html.erb`:

```erb
<main class="mcp-auth">
  <h1>Enter your code</h1>
  <p>We sent a one-time code to your Kindle as a document. Open it and type the code below.</p>

  <% if flash[:alert] %><p class="error"><%= flash[:alert] %></p><% end %>

  <%= form_with url: mcp_auth_session_verify_path, method: :post, local: true do |f| %>
    <%= f.text_field :otp, autocomplete: "one-time-code", required: true, autofocus: true %>
    <%= f.submit "Verify" %>
  <% end %>

  <%= form_with url: mcp_auth_session_resend_path, method: :post, local: true, id: "resend-form" do |f| %>
    <%= f.submit "Resend code", id: "resend-btn", disabled: true %>
  <% end %>

  <script>
    (function () {
      var remaining = <%= @resend_available_in.to_i %>;
      var btn = document.getElementById("resend-btn");
      function tick() {
        if (remaining <= 0) { btn.disabled = false; btn.value = "Resend code"; return; }
        btn.value = "Resend code (" + remaining + "s)";
        remaining -= 1;
        setTimeout(tick, 1000);
      }
      tick();
    })();
  </script>
</main>
```

- [ ] **Step 5: Run the tests, verify pass**

Run: `bundle exec rspec spec/requests/mcp/auth_sessions_verify_spec.rb`
Expected: PASS (all four examples).

- [ ] **Step 6: Commit**

```bash
git add app/controllers/mcp/auth_sessions_controller.rb app/views/mcp/auth_sessions/verify.html.erb spec/requests/mcp/auth_sessions_verify_spec.rb
git commit -m "feat(mcp): OTP verify + 30s-gated resend in auth-session flow"
```

---

### Task A6: Bind issued access tokens to the Email (resource owner)

**Files:**
- Create: `app/controllers/mcp/tokens_controller.rb`
- Test: `spec/requests/mcp/token_owner_spec.rb`

> Doorkeeper stores `resource_owner_id` on access grants/tokens. Because our `resource_owner_authenticator` returns an `Email`, `resource_owner_id` is the Email id automatically. This task adds a thin tokens controller only to confirm/lock that behavior and give us a seam for introspection payload. If the default already records `resource_owner_id`, the controller just subclasses Doorkeeper's.

- [ ] **Step 1: Write the failing test**

Create `spec/requests/mcp/token_owner_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe "Access token carries Email owner", type: :request do
  it "records resource_owner_id = Email id on the issued token" do
    email = Email.create!(email: "owner@kindle.com")
    app = Doorkeeper::Application.create!(name: "c", redirect_uri: "http://127.0.0.1/cb", confidential: false)
    grant = Doorkeeper::AccessGrant.create!(
      application: app, resource_owner_id: email.id,
      redirect_uri: "http://127.0.0.1/cb", expires_in: 600, scopes: ""
    )

    post "/oauth/token", params: {
      grant_type: "authorization_code", code: grant.token,
      client_id: app.uid, redirect_uri: "http://127.0.0.1/cb"
    }

    expect(response).to have_http_status(:ok)
    token_value = JSON.parse(response.body)["access_token"]
    token = Doorkeeper::AccessToken.by_token(token_value)
    expect(token.resource_owner_id).to eq(email.id)
  end
end
```

- [ ] **Step 2: Run it, verify it fails or passes**

Run: `bundle exec rspec spec/requests/mcp/token_owner_spec.rb`
Expected: If FAIL due to PKCE enforcement on public client, proceed to Step 3 (the test omits PKCE; relax for confidential-less grant by setting grant `code_challenge` nil path). If PASS, still add the controller in Step 3 for the introspection seam.

- [ ] **Step 3: Add the tokens controller**

Create `app/controllers/mcp/tokens_controller.rb`:

```ruby
module Mcp
  class TokensController < Doorkeeper::TokensController
    # Inherits standard token issuance. Subclassed so routes in Task A2
    # (`controllers tokens: "mcp/tokens"`) resolve, and to provide a seam
    # for future customization. No behavior change.
  end
end
```

- [ ] **Step 4: Run the test, verify pass**

Run: `bundle exec rspec spec/requests/mcp/token_owner_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add app/controllers/mcp/tokens_controller.rb spec/requests/mcp/token_owner_spec.rb
git commit -m "feat(mcp): bind issued tokens to Email resource owner"
```

---

### Task A7: Introspection returns the kindle email

**Files:**
- Modify: `config/initializers/doorkeeper.rb` (custom introspection response)
- Test: `spec/requests/mcp/introspection_spec.rb`

- [ ] **Step 1: Write the failing test**

Create `spec/requests/mcp/introspection_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe "Token introspection", type: :request do
  it "returns active=true and the kindle email for a valid token" do
    email = Email.create!(email: "owner@kindle.com")
    app = Doorkeeper::Application.create!(name: "c", redirect_uri: "http://127.0.0.1/cb", confidential: false)
    token = Doorkeeper::AccessToken.create!(
      application: app, resource_owner_id: email.id, expires_in: 3600, scopes: ""
    )

    post "/oauth/introspect", params: { token: token.token },
         headers: { "Authorization" => "Bearer #{token.token}" }

    expect(response).to have_http_status(:ok)
    body = JSON.parse(response.body)
    expect(body["active"]).to eq(true)
    expect(body["kindle_email"]).to eq("owner@kindle.com")
  end

  it "returns active=false for an unknown token" do
    email = Email.create!(email: "owner@kindle.com")
    app = Doorkeeper::Application.create!(name: "c", redirect_uri: "http://127.0.0.1/cb", confidential: false)
    valid = Doorkeeper::AccessToken.create!(application: app, resource_owner_id: email.id, expires_in: 3600, scopes: "")

    post "/oauth/introspect", params: { token: "garbage" },
         headers: { "Authorization" => "Bearer #{valid.token}" }

    expect(JSON.parse(response.body)["active"]).to eq(false)
  end
end
```

- [ ] **Step 2: Run it, verify it fails**

Run: `bundle exec rspec spec/requests/mcp/introspection_spec.rb`
Expected: FAIL (`kindle_email` absent from default introspection response).

- [ ] **Step 3: Add custom introspection response to the initializer**

In `config/initializers/doorkeeper.rb`, inside the `Doorkeeper.configure do` block, add:

```ruby
  custom_introspection_response do |token, _context|
    email = Email.find_by(id: token.resource_owner_id)
    { kindle_email: email&.email }
  end
```

- [ ] **Step 4: Run the test, verify pass**

Run: `bundle exec rspec spec/requests/mcp/introspection_spec.rb`
Expected: PASS (both examples).

- [ ] **Step 5: Commit**

```bash
git add config/initializers/doorkeeper.rb spec/requests/mcp/introspection_spec.rb
git commit -m "feat(mcp): include kindle_email in token introspection response"
```

---

### Task A8: `Mcp::SendController` — token-guarded markdown send

**Files:**
- Create: `app/controllers/mcp/send_controller.rb`
- Modify: `config/routes.rb`
- Test: `spec/requests/mcp/send_spec.rb`

- [ ] **Step 1: Write the failing tests**

Create `spec/requests/mcp/send_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe "Mcp::Send", type: :request do
  def token_for(email)
    app = Doorkeeper::Application.create!(name: "c", redirect_uri: "http://127.0.0.1/cb", confidential: false)
    Doorkeeper::AccessToken.create!(application: app, resource_owner_id: email.id, expires_in: 3600, scopes: "")
  end

  it "401s without a bearer token" do
    post "/mcp/send", params: { markdown: "# Hi", title: "Hi" }
    expect(response).to have_http_status(:unauthorized)
  end

  it "enqueues delivery to the token's kindle email and returns ok" do
    email = Email.create!(email: "reader@kindle.com")
    token = token_for(email)

    expect {
      post "/mcp/send",
           params: { markdown: "# Hello\n\nBody", title: "Hello", author: "Jane" },
           headers: { "Authorization" => "Bearer #{token.token}" }
    }.to have_enqueued_job(DeliveryJob)

    expect(response).to have_http_status(:ok)
    article = email.articles.last
    expect(article.markdown).to include("Hello")
    expect(article.title).to eq("Hello")
    expect(article.author).to eq("Jane")
  end

  it "defaults author to the email local-part when omitted" do
    email = Email.create!(email: "reader@kindle.com")
    token = token_for(email)
    post "/mcp/send", params: { markdown: "# X", title: "X" },
         headers: { "Authorization" => "Bearer #{token.token}" }
    expect(email.articles.last.author).to eq("reader")
  end

  it "400s on empty markdown" do
    email = Email.create!(email: "reader@kindle.com")
    token = token_for(email)
    post "/mcp/send", params: { markdown: "  ", title: "X" },
         headers: { "Authorization" => "Bearer #{token.token}" }
    expect(response).to have_http_status(:bad_request)
  end

  it "429s when the monthly limit is reached" do
    email = Email.create!(email: "reader@kindle.com")
    allow_any_instance_of(Email).to receive(:articles_in_period).and_return(9999)
    allow_any_instance_of(Email).to receive(:max_articles_per_month).and_return(10)
    token = token_for(email)
    post "/mcp/send", params: { markdown: "# X", title: "X" },
         headers: { "Authorization" => "Bearer #{token.token}" }
    expect(response).to have_http_status(:too_many_requests)
  end
end
```

- [ ] **Step 2: Run it, verify it fails**

Run: `bundle exec rspec spec/requests/mcp/send_spec.rb`
Expected: FAIL (no `/mcp/send` route).

- [ ] **Step 3: Add the route**

In `config/routes.rb`:

```ruby
post "/mcp/send", to: "mcp/send#create"
```

- [ ] **Step 4: Implement the controller**

Create `app/controllers/mcp/send_controller.rb`:

```ruby
module Mcp
  class SendController < ActionController::API
    include Doorkeeper::Rails::Helpers
    before_action -> { doorkeeper_authorize! }

    def create
      markdown = params[:markdown].to_s
      if markdown.strip.empty?
        return render json: { error: "missing_markdown" }, status: :bad_request
      end

      email_record = current_email
      return render json: { error: "unknown_owner" }, status: :unauthorized unless email_record

      if email_record.articles_in_period >= email_record.max_articles_per_month
        return render json: { error: "limit_reached" }, status: :too_many_requests
      end

      article = email_record.articles.create!(
        url: "",
        title: params[:title].to_s,
        author: params[:author].presence || email_record.email.split("@").first,
        markdown: markdown,
        sent_status: :scheduled
      )

      DeliveryJob.perform_later(article.id, email_record.email)
      render json: { status: "ok", message: "Sending to #{email_record.email}" }
    end

    private

    def current_email
      Email.find_by(id: doorkeeper_token&.resource_owner_id)
    end
  end
end
```

- [ ] **Step 5: Run the tests, verify pass**

Run: `bundle exec rspec spec/requests/mcp/send_spec.rb`
Expected: PASS (all five examples).

- [ ] **Step 6: Commit**

```bash
git add app/controllers/mcp/send_controller.rb config/routes.rb spec/requests/mcp/send_spec.rb
git commit -m "feat(mcp): token-guarded markdown send endpoint reusing delivery pipeline"
```

---

### Task A9: Full Phase A suite + DeliveryJob reuse check

- [ ] **Step 1: Run the whole MCP request suite**

Run: `bundle exec rspec spec/requests/mcp`
Expected: all examples PASS.

- [ ] **Step 2: Run the existing suite to confirm no regressions**

Run: `bundle exec rspec`
Expected: green (or no new failures vs. baseline).

- [ ] **Step 3: Manual smoke (optional, documented)**

Boot the app and confirm `GET /.well-known/oauth-authorization-server` returns JSON and `POST /oauth/register` returns a `client_id`. Document the commands in the PR description.

- [ ] **Step 4: Open PR for the worktree branch**

```bash
git push -u origin mcp-oauth
gh pr create --title "MCP OAuth 2.1 Authorization Server + Kindle send endpoint" \
  --body "Implements the AS, DCR, OTP login (with 30s resend), introspection, and /mcp/send. Spec: docs/superpowers/specs/2026-06-06-mcp-kindle-oauth-design.md"
```

---

## PHASE B — readitsoon-mcp (TypeScript MCP Resource Server)

> This is the current repo. Commit to `main`, atomic commits.

### Task B1: Scaffold the TypeScript project

**Files:**
- Create: `package.json`, `tsconfig.json`, `.gitignore`, `vitest.config.ts`, `src/config.ts`

- [ ] **Step 1: Create `package.json`**

```json
{
  "name": "readitsoon-mcp",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "bin": { "readitsoon-mcp": "dist/index.js" },
  "scripts": {
    "build": "tsc",
    "start": "node dist/index.js",
    "dev": "tsx src/index.ts",
    "test": "vitest run",
    "test:watch": "vitest"
  },
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.12.0",
    "express": "^4.19.2",
    "zod": "^3.23.8"
  },
  "devDependencies": {
    "@types/express": "^4.17.21",
    "@types/node": "^20.14.0",
    "tsx": "^4.16.0",
    "typescript": "^5.5.0",
    "vitest": "^2.0.0"
  }
}
```

- [ ] **Step 2: Create `tsconfig.json`**

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "Node16",
    "moduleResolution": "Node16",
    "outDir": "dist",
    "rootDir": "src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "declaration": false,
    "resolveJsonModule": true
  },
  "include": ["src"]
}
```

- [ ] **Step 3: Create `.gitignore`**

```
node_modules
dist
*.log
.env
```

- [ ] **Step 4: Create `vitest.config.ts`**

```ts
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: { environment: "node", include: ["src/**/*.test.ts"] }
});
```

- [ ] **Step 5: Create `src/config.ts`**

```ts
export interface Config {
  port: number;
  /** Public URL of THIS MCP resource server, e.g. https://mcp.readitsoon.com */
  resourceServerUrl: string;
  /** readitsoon OAuth issuer base URL, e.g. https://readitsoon.com */
  authServerUrl: string;
}

export function loadConfig(env: NodeJS.ProcessEnv = process.env): Config {
  const resourceServerUrl = env.MCP_RESOURCE_URL ?? "http://localhost:3030";
  const authServerUrl = env.READITSOON_URL ?? "http://localhost:3000";
  const port = Number(env.PORT ?? 3030);
  return { port, resourceServerUrl, authServerUrl };
}
```

- [ ] **Step 6: Install + verify build**

Run: `npm install && npm run build`
Expected: installs, `tsc` exits 0 (no `src/index.ts` yet → create empty `src/index.ts` with `export {};` if build complains; it will be replaced in B5).

- [ ] **Step 7: Commit**

```bash
git add package.json package-lock.json tsconfig.json .gitignore vitest.config.ts src/config.ts
git commit -m "chore: scaffold readitsoon-mcp TypeScript project"
```

---

### Task B2: Introspection-backed token verifier

**Files:**
- Create: `src/introspection-verifier.ts`
- Test: `src/introspection-verifier.test.ts`

> Implements the SDK's `OAuthTokenVerifier` interface: `verifyAccessToken(token) => Promise<AuthInfo>`. `AuthInfo` requires `{ token, clientId, scopes, expiresAt?, extra? }`. We call readitsoon `/oauth/introspect` and put `kindle_email` in `extra`.

- [ ] **Step 1: Write the failing test**

Create `src/introspection-verifier.test.ts`:

```ts
import { describe, it, expect, vi } from "vitest";
import { IntrospectionVerifier } from "./introspection-verifier.js";

function fetchReturning(status: number, body: unknown) {
  return vi.fn(async () => ({
    ok: status < 400,
    status,
    json: async () => body
  })) as unknown as typeof fetch;
}

describe("IntrospectionVerifier", () => {
  it("returns AuthInfo with kindle_email in extra for an active token", async () => {
    const f = fetchReturning(200, { active: true, kindle_email: "r@kindle.com", exp: 9999999999 });
    const v = new IntrospectionVerifier("https://readitsoon.test", f);
    const info = await v.verifyAccessToken("abc");
    expect(info.token).toBe("abc");
    expect(info.extra?.kindleEmail).toBe("r@kindle.com");
  });

  it("throws for an inactive token", async () => {
    const f = fetchReturning(200, { active: false });
    const v = new IntrospectionVerifier("https://readitsoon.test", f);
    await expect(v.verifyAccessToken("abc")).rejects.toThrow();
  });

  it("caches a positive result to avoid a second network call", async () => {
    const f = fetchReturning(200, { active: true, kindle_email: "r@kindle.com" });
    const v = new IntrospectionVerifier("https://readitsoon.test", f);
    await v.verifyAccessToken("abc");
    await v.verifyAccessToken("abc");
    expect((f as any).mock.calls.length).toBe(1);
  });
});
```

- [ ] **Step 2: Run it, verify it fails**

Run: `npm test -- introspection-verifier`
Expected: FAIL (module missing).

- [ ] **Step 3: Implement the verifier**

Create `src/introspection-verifier.ts`:

```ts
import type { OAuthTokenVerifier } from "@modelcontextprotocol/sdk/server/auth/provider.js";
import type { AuthInfo } from "@modelcontextprotocol/sdk/server/auth/types.js";

interface CacheEntry { info: AuthInfo; expiresAtMs: number; }

export class IntrospectionVerifier implements OAuthTokenVerifier {
  private cache = new Map<string, CacheEntry>();
  private readonly cacheTtlMs = 30_000;

  constructor(
    private readonly authServerUrl: string,
    private readonly fetchImpl: typeof fetch = fetch
  ) {}

  async verifyAccessToken(token: string): Promise<AuthInfo> {
    const cached = this.cache.get(token);
    if (cached && cached.expiresAtMs > Date.now()) return cached.info;

    const res = await this.fetchImpl(`${this.authServerUrl}/oauth/introspect`, {
      method: "POST",
      headers: {
        "Content-Type": "application/x-www-form-urlencoded",
        Authorization: `Bearer ${token}`
      },
      body: new URLSearchParams({ token }).toString()
    });

    if (!res.ok) throw new Error(`introspection_failed_${res.status}`);
    const data = (await res.json()) as {
      active?: boolean; kindle_email?: string; exp?: number; scope?: string; client_id?: string;
    };
    if (!data.active) throw new Error("invalid_token");

    const info: AuthInfo = {
      token,
      clientId: data.client_id ?? "unknown",
      scopes: data.scope ? data.scope.split(" ") : [],
      expiresAt: data.exp,
      extra: { kindleEmail: data.kindle_email }
    };

    this.cache.set(token, { info, expiresAtMs: Date.now() + this.cacheTtlMs });
    return info;
  }
}
```

- [ ] **Step 4: Run the test, verify pass**

Run: `npm test -- introspection-verifier`
Expected: PASS (all three examples).

- [ ] **Step 5: Commit**

```bash
git add src/introspection-verifier.ts src/introspection-verifier.test.ts
git commit -m "feat: introspection-backed OAuth token verifier with cache"
```

---

### Task B3: The send-to-readitsoon client

**Files:**
- Create: `src/readitsoon-client.ts`
- Test: `src/readitsoon-client.test.ts`

- [ ] **Step 1: Write the failing test**

Create `src/readitsoon-client.test.ts`:

```ts
import { describe, it, expect, vi } from "vitest";
import { sendMarkdown } from "./readitsoon-client.js";

describe("sendMarkdown", () => {
  it("POSTs markdown/title/author with the bearer token and returns the message", async () => {
    const f = vi.fn(async () => ({
      ok: true, status: 200, json: async () => ({ status: "ok", message: "Sending to r@kindle.com" })
    })) as unknown as typeof fetch;

    const msg = await sendMarkdown(
      "https://readitsoon.test", "tok-123",
      { markdown: "# Hi", title: "Hi", author: "Jane" }, f
    );

    expect(msg).toBe("Sending to r@kindle.com");
    const [url, opts] = (f as any).mock.calls[0];
    expect(url).toBe("https://readitsoon.test/mcp/send");
    expect((opts.headers as Record<string, string>).Authorization).toBe("Bearer tok-123");
    const sent = JSON.parse(opts.body as string);
    expect(sent).toMatchObject({ markdown: "# Hi", title: "Hi", author: "Jane" });
  });

  it("throws with the server error message on non-2xx", async () => {
    const f = vi.fn(async () => ({
      ok: false, status: 429, json: async () => ({ error: "limit_reached" })
    })) as unknown as typeof fetch;

    await expect(
      sendMarkdown("https://readitsoon.test", "t", { markdown: "x", title: "x" }, f)
    ).rejects.toThrow("limit_reached");
  });
});
```

- [ ] **Step 2: Run it, verify it fails**

Run: `npm test -- readitsoon-client`
Expected: FAIL (module missing).

- [ ] **Step 3: Implement the client**

Create `src/readitsoon-client.ts`:

```ts
export interface SendParams {
  markdown: string;
  title: string;
  author?: string;
}

export async function sendMarkdown(
  authServerUrl: string,
  bearerToken: string,
  params: SendParams,
  fetchImpl: typeof fetch = fetch
): Promise<string> {
  const res = await fetchImpl(`${authServerUrl}/mcp/send`, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      Authorization: `Bearer ${bearerToken}`
    },
    body: JSON.stringify(params)
  });

  const data = (await res.json()) as { status?: string; message?: string; error?: string };
  if (!res.ok) throw new Error(data.error ?? `send_failed_${res.status}`);
  return data.message ?? "Sent.";
}
```

- [ ] **Step 4: Run the test, verify pass**

Run: `npm test -- readitsoon-client`
Expected: PASS (both examples).

- [ ] **Step 5: Commit**

```bash
git add src/readitsoon-client.ts src/readitsoon-client.test.ts
git commit -m "feat: readitsoon /mcp/send client"
```

---

### Task B4: The `send_markdown_to_kindle` tool factory

**Files:**
- Create: `src/tool.ts`
- Test: `src/tool.test.ts`

> The tool reads the authenticated bearer token + kindle email from the request's `AuthInfo` (the SDK populates `extra.authInfo` on the tool-call request handler `extra` argument). It derives the EPUB title from `filename` (basename, strip extension) and passes `author` through (Claude infers it from the markdown; if omitted, readitsoon defaults to the email local-part).

- [ ] **Step 1: Write the failing test**

Create `src/tool.test.ts`:

```ts
import { describe, it, expect, vi } from "vitest";
import { buildSendHandler, titleFromFilename } from "./tool.js";

describe("titleFromFilename", () => {
  it("strips directory and extension", () => {
    expect(titleFromFilename("/notes/My Great Doc.md")).toBe("My Great Doc");
    expect(titleFromFilename("plain")).toBe("plain");
  });
});

describe("buildSendHandler", () => {
  const authExtra = (token: string) => ({
    authInfo: { token, clientId: "c", scopes: [], extra: { kindleEmail: "r@kindle.com" } }
  });

  it("calls the sender with token, title from filename, and author", async () => {
    const sender = vi.fn(async () => "Sending to r@kindle.com");
    const handler = buildSendHandler(sender);

    const result = await handler(
      { markdown: "# Hi\n\nby Jane", filename: "notes/Hi.md", author: "Jane" },
      authExtra("tok-9") as any
    );

    expect(sender).toHaveBeenCalledWith("tok-9", { markdown: "# Hi\n\nby Jane", title: "Hi", author: "Jane" });
    expect(result.content[0]).toMatchObject({ type: "text", text: "Sending to r@kindle.com" });
  });

  it("returns an error result (isError) when the sender throws", async () => {
    const sender = vi.fn(async () => { throw new Error("limit_reached"); });
    const handler = buildSendHandler(sender);
    const result = await handler({ markdown: "x", filename: "x.md" }, authExtra("t") as any);
    expect(result.isError).toBe(true);
    expect(result.content[0].text).toContain("limit_reached");
  });
});
```

- [ ] **Step 2: Run it, verify it fails**

Run: `npm test -- tool`
Expected: FAIL (module missing).

- [ ] **Step 3: Implement the tool factory**

Create `src/tool.ts`:

```ts
import { z } from "zod";

export const sendToolInputShape = {
  markdown: z.string().min(1).describe("The raw Markdown content to send to the Kindle."),
  filename: z.string().min(1).describe("The source filename; its basename (without extension) becomes the EPUB title."),
  author: z.string().optional().describe(
    "Author for the EPUB. Infer this from the Markdown (frontmatter, byline, or title block) when present. If omitted, the server defaults to the account's email name."
  )
};

export interface SendArgs { markdown: string; filename: string; author?: string; }

export type Sender = (token: string, params: { markdown: string; title: string; author?: string }) => Promise<string>;

export function titleFromFilename(filename: string): string {
  const base = filename.split("/").pop() ?? filename;
  const dot = base.lastIndexOf(".");
  return dot > 0 ? base.slice(0, dot) : base;
}

export function buildSendHandler(sender: Sender) {
  return async (args: SendArgs, extra: { authInfo?: { token: string } }) => {
    const token = extra.authInfo?.token;
    if (!token) {
      return { isError: true, content: [{ type: "text" as const, text: "Not authenticated." }] };
    }
    try {
      const message = await sender(token, {
        markdown: args.markdown,
        title: titleFromFilename(args.filename),
        author: args.author
      });
      return { content: [{ type: "text" as const, text: message }] };
    } catch (err) {
      const msg = err instanceof Error ? err.message : String(err);
      return { isError: true, content: [{ type: "text" as const, text: `Failed to send: ${msg}` }] };
    }
  };
}
```

- [ ] **Step 4: Run the test, verify pass**

Run: `npm test -- tool`
Expected: PASS (all examples).

- [ ] **Step 5: Commit**

```bash
git add src/tool.ts src/tool.test.ts
git commit -m "feat: send_markdown_to_kindle tool factory"
```

---

### Task B5: Wire the Express server (transport + auth + metadata + tool)

**Files:**
- Create: `src/server.ts`
- Create: `src/index.ts`
- Test: `src/server.test.ts`

> Uses `McpServer` + `StreamableHTTPServerTransport`, guards `/mcp` with `requireBearerAuth({ verifier })`, and serves protected-resource metadata with `mcpAuthMetadataRouter`. Import paths follow `@modelcontextprotocol/sdk` v1.x.

- [ ] **Step 1: Write the failing test (unauthenticated /mcp → 401)**

Create `src/server.test.ts`:

```ts
import { describe, it, expect } from "vitest";
import request from "node:http";
import { createApp } from "./server.js";
import { IntrospectionVerifier } from "./introspection-verifier.js";

// Minimal supertest-free check using the app's handler via a real listen.
import { createServer } from "node:http";

async function get(app: any, path: string): Promise<{ status: number; body: string }> {
  const server = createServer(app);
  await new Promise<void>((r) => server.listen(0, r));
  const { port } = server.address() as any;
  const res: { status: number; body: string } = await new Promise((resolve, reject) => {
    request.get({ host: "127.0.0.1", port, path }, (r) => {
      let data = ""; r.on("data", (c) => (data += c));
      r.on("end", () => resolve({ status: r.statusCode ?? 0, body: data }));
    }).on("error", reject);
  });
  await new Promise<void>((r) => server.close(() => r()));
  return res;
}

describe("createApp", () => {
  const verifier = new IntrospectionVerifier("http://auth.test", (async () => ({
    ok: true, status: 200, json: async () => ({ active: false })
  })) as unknown as typeof fetch);

  const app = createApp({
    config: { port: 0, resourceServerUrl: "http://localhost:3030", authServerUrl: "http://auth.test" },
    verifier,
    sender: async () => "ok"
  });

  it("serves protected-resource metadata", async () => {
    const res = await get(app, "/.well-known/oauth-protected-resource");
    expect(res.status).toBe(200);
    const body = JSON.parse(res.body);
    expect(body.authorization_servers).toContain("http://auth.test");
  });

  it("401s on POST /mcp without a token", async () => {
    const res = await get(app, "/mcp"); // GET on protected route still requires auth
    expect(res.status).toBe(401);
  });
});
```

- [ ] **Step 2: Run it, verify it fails**

Run: `npm test -- server`
Expected: FAIL (module missing).

- [ ] **Step 3: Implement `src/server.ts`**

```ts
import express, { type Express } from "express";
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js";
import { requireBearerAuth } from "@modelcontextprotocol/sdk/server/auth/middleware/bearerAuth.js";
import { mcpAuthMetadataRouter } from "@modelcontextprotocol/sdk/server/auth/router.js";
import type { OAuthMetadata } from "@modelcontextprotocol/sdk/shared/auth.js";
import type { OAuthTokenVerifier } from "@modelcontextprotocol/sdk/server/auth/provider.js";
import { sendToolInputShape, buildSendHandler, type Sender } from "./tool.js";
import type { Config } from "./config.js";

export interface AppDeps {
  config: Config;
  verifier: OAuthTokenVerifier;
  sender: Sender;
}

function buildMcpServer(deps: AppDeps): McpServer {
  const server = new McpServer({ name: "readitsoon-mcp", version: "0.1.0" });
  const handler = buildSendHandler(deps.sender);
  server.registerTool(
    "send_markdown_to_kindle",
    {
      title: "Send Markdown to Kindle",
      description: "Send a Markdown document to the authenticated user's Kindle as an EPUB.",
      inputSchema: sendToolInputShape
    },
    handler as any
  );
  return server;
}

export function createApp(deps: AppDeps): Express {
  const app = express();
  app.use(express.json({ limit: "5mb" }));

  const oauthMetadata: OAuthMetadata = {
    issuer: deps.config.authServerUrl,
    authorization_endpoint: `${deps.config.authServerUrl}/oauth/authorize`,
    token_endpoint: `${deps.config.authServerUrl}/oauth/token`,
    registration_endpoint: `${deps.config.authServerUrl}/oauth/register`,
    introspection_endpoint: `${deps.config.authServerUrl}/oauth/introspect`,
    response_types_supported: ["code"],
    grant_types_supported: ["authorization_code", "refresh_token"],
    code_challenge_methods_supported: ["S256"]
  };

  app.use(
    mcpAuthMetadataRouter({
      oauthMetadata,
      resourceServerUrl: new URL(deps.config.resourceServerUrl),
      scopesSupported: [],
      resourceName: "Read It Soon Kindle Sender"
    })
  );

  const bearer = requireBearerAuth({ verifier: deps.verifier });

  app.all("/mcp", bearer, async (req, res) => {
    const transport = new StreamableHTTPServerTransport({ sessionIdGenerator: undefined });
    const server = buildMcpServer(deps);
    await server.connect(transport);
    await transport.handleRequest(req, res, req.body);
  });

  return app;
}
```

- [ ] **Step 4: Implement `src/index.ts` (entrypoint)**

```ts
import { loadConfig } from "./config.js";
import { createApp } from "./server.js";
import { IntrospectionVerifier } from "./introspection-verifier.js";
import { sendMarkdown } from "./readitsoon-client.js";

const config = loadConfig();
const verifier = new IntrospectionVerifier(config.authServerUrl);
const sender = (token: string, params: { markdown: string; title: string; author?: string }) =>
  sendMarkdown(config.authServerUrl, token, params);

const app = createApp({ config, verifier, sender });
app.listen(config.port, () => {
  console.log(`readitsoon-mcp listening on :${config.port}`);
});
```

- [ ] **Step 5: Run the test, verify pass**

Run: `npm test -- server`
Expected: PASS. If the SDK rejects a `GET /mcp` differently than `POST`, the 401 assertion still holds because `requireBearerAuth` runs before the transport. If import paths differ in the installed SDK version, run `node -e "console.log(require.resolve('@modelcontextprotocol/sdk/server/auth/router.js'))"` and adjust the import specifiers to the actual files under `node_modules/@modelcontextprotocol/sdk/dist/esm/...`.

- [ ] **Step 6: Build**

Run: `npm run build`
Expected: `tsc` exits 0.

- [ ] **Step 7: Commit**

```bash
git add src/server.ts src/index.ts src/server.test.ts
git commit -m "feat: wire MCP streamable-HTTP server with bearer auth + metadata + tool"
```

---

### Task B6: Full suite, README, and end-to-end smoke

**Files:**
- Create: `README.md`

- [ ] **Step 1: Run the full test suite**

Run: `npm test`
Expected: all suites PASS.

- [ ] **Step 2: Write `README.md`**

Create `README.md`:

```markdown
# readitsoon-mcp

Remote HTTP MCP server exposing `send_markdown_to_kindle`. Acts as an OAuth 2.1
Resource Server; readitsoon is the Authorization Server.

## Env
- `PORT` (default 3030)
- `MCP_RESOURCE_URL` — public URL of this server (e.g. https://mcp.readitsoon.com)
- `READITSOON_URL` — readitsoon base URL / OAuth issuer

## Run
    npm install && npm run build && npm start

## Connect from Claude Code
Add the remote server URL (…/mcp). Claude Code performs DCR + OAuth automatically;
log in with your Kindle email and the one-time code delivered to your device.

## Tool
`send_markdown_to_kindle(markdown, filename, author?)` — title comes from `filename`,
the Kindle address comes from your authenticated token (not a parameter).
```

- [ ] **Step 3: End-to-end smoke (documented, requires Phase A running locally)**

With readitsoon running on `:3000` and this server on `:3030`:
1. `curl http://localhost:3030/.well-known/oauth-protected-resource` → JSON pointing at `:3000`.
2. `curl -i http://localhost:3030/mcp` → `401` with `WWW-Authenticate`.
3. Connect from Claude Code, complete OAuth, run "send this markdown to my kindle" on a `.md` file → EPUB arrives.

Document outcomes in the commit/PR.

- [ ] **Step 4: Commit**

```bash
git add README.md
git commit -m "docs: add readitsoon-mcp README"
```

---

## Self-Review Notes (resolved)

- **Spec coverage:** transport (B5), AS roles (A2/B5), opaque+introspection (A7/B2), DCR (A3), single-email identity (A6/A7/A8), OTP-epub + whitelist step (A4 view), resend 30s (A5), Doorkeeper+shims (A1–A7), reuse map (A4/A8), tool shape incl. author inference (B4), error handling (A8/B2/B3/B4), testing (every task). All covered.
- **Type consistency:** `kindle_email` (Rails JSON) ↔ `kindleEmail` (`extra`) mapping is intentional and done in B2. `Sender` signature `(token, {markdown,title,author?})` consistent across B3/B4/B5. `titleFromFilename` used only in B4.
- **Known version risk:** `@modelcontextprotocol/sdk` import specifiers (B5 Step 3) and `custom_introspection_response` / `allow_token_introspection` Doorkeeper DSL (A2/A7) are the two spots most likely to need a small adjustment against the installed versions; each has an inline fallback instruction.
