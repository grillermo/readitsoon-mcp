# MCP Kindle Send + OAuth 2.1 Implementation Plan (Rails-only)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** readitsoon serves a remote Streamable-HTTP MCP endpoint with a `send_markdown_to_kindle` tool, guarded by an OAuth 2.1 Authorization Server that readitsoon also hosts, reusing the existing markdown→EPUB→Kindle pipeline.

**Architecture:** Everything lives in **readitsoon** (Rails). The MCP Resource Server and the OAuth Authorization Server are the same app, so access tokens are validated **locally** via Doorkeeper `doorkeeper_authorize!` — no introspection endpoint, no second host. MCP protocol is provided by the official `mcp` gem (`modelcontextprotocol/ruby-sdk`): the tool is an `MCP::Tool` subclass, the endpoint is `Mcp::ServerController` mounting `StreamableHTTPTransport` (`stateless: true`). New code is under the `Mcp::` namespace. The `readitsoon-mcp` repo holds only spec/plan/E2E fixtures.

**Tech Stack:** Ruby on Rails, Doorkeeper, `mcp` gem, RSpec, Mailgun, Pandoc (existing).

**Local URLs:** Rails runs on **`http://localhost:4001`** (`RAILS_PORT=4001`). MCP endpoint: `http://localhost:4001/mcp`. OAuth issuer is derived from `request.base_url` (→ `http://localhost:4001`). Approved-sender to whitelist for OTP delivery: **`sending@readitsoon.app`** (`SENDER_EMAIL`).

**Repo / branching:** All implementation is in **readitsoon**, in a git worktree under `../readitsoon/.worktrees/mcp-oauth` on branch `mcp-oauth`. Create it at execution start via `superpowers:using-git-worktrees`. Atomic commits on that branch. Spec/plan changes commit to `readitsoon-mcp` `main`.

**Spec:** `docs/superpowers/specs/2026-06-06-mcp-kindle-oauth-design.md`

**Reuse map (readitsoon, do not reimplement):**
- `EpubCreator.convert(markdown, title, author, url)` → `[epub_path, dir]`
- `DeliveryService.call(epub_path, dir, url, title, recipient)`
- `DeliveryJob.perform_later(article_id, recipient)` (calls EpubCreator + DeliveryService)
- `Email#issue_sign_in_otp!` → otp string; `Email#valid_sign_in_otp?(otp)`; `Email#consume_sign_in_otp!`; `Email#token`; `Email#articles`; `Email#articles_in_period`; `Email#max_articles_per_month`
- `OtpSignInDeliveryService.call(email_record, otp)` (sends OTP as epub)
- `SignInIpGuard.new(request.remote_ip)` → `#banned?`, `#register_failure!`, `#reset_failures!`
- `MailgunEmailClient` — the class to instrument with a STDOUT delivery log (Task 6)

---

### Task 1: Install Doorkeeper + the mcp gem

**Files:**
- Modify: `Gemfile`
- Create: `config/initializers/doorkeeper.rb`, `db/migrate/*_create_doorkeeper_tables.rb` (generated)
- Modify: `config/routes.rb`

- [ ] **Step 1: Add gems**

Add to `Gemfile` near `gem "devise"`:

```ruby
gem "doorkeeper", "~> 5.8"
gem "mcp"
```

- [ ] **Step 2: Install**

Run: `bundle install`
Expected: resolves; installs `doorkeeper` 5.8.x and `mcp`.

- [ ] **Step 3: Generate Doorkeeper install + migration**

Run: `bin/rails generate doorkeeper:install && bin/rails generate doorkeeper:migration`
Expected: creates `config/initializers/doorkeeper.rb`, adds `use_doorkeeper` to routes, creates the migration.

- [ ] **Step 4: Make the migration support public clients + PKCE**

In the generated `*_create_doorkeeper_tables.rb`:
- `oauth_applications`: set `t.boolean :confidential, null: false, default: false` (public clients). Keep `t.string :secret`.
- `oauth_access_grants`: uncomment `t.string :code_challenge` and `t.string :code_challenge_method`.

- [ ] **Step 5: Migrate**

Run: `bin/rails db:migrate`
Expected: `oauth_applications`, `oauth_access_grants`, `oauth_access_tokens` created.

- [ ] **Step 6: Commit**

```bash
git add Gemfile Gemfile.lock config/initializers/doorkeeper.rb config/routes.rb db/migrate db/schema.rb
git commit -m "feat(mcp): install doorkeeper (PKCE, public clients) and mcp gem"
```

---

### Task 2: Configure Doorkeeper + serve AS and Protected-Resource metadata

**Files:**
- Modify: `config/initializers/doorkeeper.rb`, `config/routes.rb`
- Create: `app/controllers/mcp/metadata_controller.rb`
- Test: `spec/requests/mcp/metadata_spec.rb`

- [ ] **Step 1: Write the failing test**

Create `spec/requests/mcp/metadata_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe "OAuth + MCP metadata", type: :request do
  it "serves RFC 8414 authorization-server metadata" do
    get "/.well-known/oauth-authorization-server"
    expect(response).to have_http_status(:ok)
    body = JSON.parse(response.body)
    expect(body["issuer"]).to be_present
    expect(body["authorization_endpoint"]).to include("/oauth/authorize")
    expect(body["token_endpoint"]).to include("/oauth/token")
    expect(body["registration_endpoint"]).to include("/oauth/register")
    expect(body["code_challenge_methods_supported"]).to include("S256")
  end

  it "serves RFC 9728 protected-resource metadata pointing at the AS" do
    get "/.well-known/oauth-protected-resource"
    expect(response).to have_http_status(:ok)
    body = JSON.parse(response.body)
    expect(body["resource"]).to include("/mcp")
    expect(body["authorization_servers"].first).to eq(body["resource"].sub("/mcp", ""))
  end
end
```

- [ ] **Step 2: Run it, verify it fails**

Run: `bundle exec rspec spec/requests/mcp/metadata_spec.rb`
Expected: FAIL (routes missing).

- [ ] **Step 3: Configure the initializer**

Replace the `Doorkeeper.configure do ... end` body in `config/initializers/doorkeeper.rb` with:

```ruby
Doorkeeper.configure do
  orm :active_record

  # Custom login: hand control to our OTP auth-session flow.
  resource_owner_authenticator do
    if session[:mcp_authenticated_email_id]
      Email.find_by(id: session[:mcp_authenticated_email_id])
    else
      redirect_to(mcp_auth_session_path(return_to: request.fullpath))
      nil
    end
  end

  # Public native/desktop clients: no secret, PKCE required.
  force_pkce
  grant_flows %w[authorization_code]
  access_token_expires_in 2.hours
  use_refresh_token

  # No admin UI.
  base_controller "ActionController::Base"
end
```

- [ ] **Step 4: Add routes**

In `config/routes.rb`, keep `use_doorkeeper` (default), and add:

```ruby
get "/.well-known/oauth-authorization-server", to: "mcp/metadata#authorization_server"
get "/.well-known/oauth-protected-resource",   to: "mcp/metadata#protected_resource"
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
        registration_endpoint: "#{issuer}/oauth/register",
        response_types_supported: ["code"],
        grant_types_supported: %w[authorization_code refresh_token],
        code_challenge_methods_supported: ["S256"],
        token_endpoint_auth_methods_supported: ["none"]
      }
    end

    def protected_resource
      issuer = request.base_url
      render json: {
        resource: "#{issuer}/mcp",
        authorization_servers: [issuer],
        scopes_supported: [],
        resource_name: "Read It Soon Kindle Sender"
      }
    end
  end
end
```

- [ ] **Step 6: Run the test, verify pass**

Run: `bundle exec rspec spec/requests/mcp/metadata_spec.rb`
Expected: PASS (both examples).

- [ ] **Step 7: Commit**

```bash
git add config/initializers/doorkeeper.rb config/routes.rb app/controllers/mcp/metadata_controller.rb spec/requests/mcp/metadata_spec.rb
git commit -m "feat(mcp): configure doorkeeper AS + serve AS/protected-resource metadata"
```

---

### Task 3: Dynamic Client Registration (RFC 7591)

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

  it "rejects non-loopback redirect_uris" do
    post "/oauth/register", params: {
      client_name: "Bad Client",
      redirect_uris: ["https://example.com/callback"],
      token_endpoint_auth_method: "none"
    }.to_json, headers: { "Content-Type" => "application/json" }

    expect(response).to have_http_status(:bad_request)
    expect(JSON.parse(response.body)["error"]).to eq("invalid_redirect_uri")
  end
end
```

- [ ] **Step 2: Run it, verify it fails**

Run: `bundle exec rspec spec/requests/mcp/registrations_spec.rb`
Expected: FAIL (no `/oauth/register`).

- [ ] **Step 3: Add the route**

In `config/routes.rb`:

```ruby
post "/oauth/register", to: "mcp/registrations#create"
```

- [ ] **Step 4: Implement the controller**

Create `app/controllers/mcp/registrations_controller.rb`:

```ruby
require "uri"

module Mcp
  class RegistrationsController < ActionController::API
    LOOPBACK_HOSTS = %w[localhost 127.0.0.1 ::1].freeze

    def create
      redirect_uris = Array(params[:redirect_uris]).map(&:to_s).reject(&:blank?)
      if redirect_uris.empty? || redirect_uris.any? { |uri| !loopback_redirect_uri?(uri) }
        return render json: { error: "invalid_redirect_uri",
                              error_description: "redirect_uris must be loopback HTTP /callback URLs" },
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

    private

    def loopback_redirect_uri?(uri)
      parsed = URI.parse(uri)
      parsed.scheme == "http" &&
        LOOPBACK_HOSTS.include?(parsed.host) &&
        parsed.path == "/callback"
    rescue URI::InvalidURIError
      false
    end
  end
end
```

- [ ] **Step 5: Run the test, verify pass**

Run: `bundle exec rspec spec/requests/mcp/registrations_spec.rb`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add app/controllers/mcp/registrations_controller.rb config/routes.rb spec/requests/mcp/registrations_spec.rb
git commit -m "feat(mcp): add RFC 7591 dynamic client registration endpoint"
```

---

### Task 4: OTP auth-session — email entry + send OTP (with dev STDOUT echo)

**Files:**
- Create: `app/controllers/mcp/auth_sessions_controller.rb`, `app/views/mcp/auth_sessions/new.html.erb`
- Modify: `config/routes.rb`, `app/models/email.rb` (dev STDOUT echo of OTP)
- Test: `spec/requests/mcp/auth_sessions_spec.rb`

- [ ] **Step 1: Write the failing test**

Create `spec/requests/mcp/auth_sessions_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe "MCP auth session", type: :request do
  it "shows the email entry form mentioning the whitelist step" do
    get "/mcp/auth_session", params: { return_to: "/oauth/authorize?x=1" }
    expect(response).to have_http_status(:ok)
    expect(response.body.downcase).to include("kindle")
    expect(response.body).to include("sending@readitsoon.app")
  end

  it "issues an OTP epub and advances to the verify step" do
    expect(OtpSignInDeliveryService).to receive(:call).once
    post "/mcp/auth_session", params: { email: "reader@kindle.com", return_to: "/oauth/authorize?x=1" }
    expect(response).to redirect_to(mcp_auth_session_verify_path)
    email = Email.find_by(email: "reader@kindle.com")
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

- [ ] **Step 4: Implement `new` + `create` (rest stubbed for Task 5)**

Create `app/controllers/mcp/auth_sessions_controller.rb`:

```ruby
module Mcp
  class AuthSessionsController < ApplicationController
    skip_before_action :track_ahoy_visit, raise: false
    skip_around_action :set_ahoy_request_store, raise: false
    skip_forgery_protection

    layout "application"

    SENDER_EMAIL = ENV.fetch("SENDER_EMAIL", "sending@readitsoon.app")
    RESEND_INTERVAL = 30.seconds

    def new
      session[:mcp_return_to] = params[:return_to] if params[:return_to].present?
      @sender_email = SENDER_EMAIL
      @email = ""
    end

    def create
      return render_ip_banned if ip_guard.banned?

      email_record = Email.find_or_initialize_by(email: email_param)
      unless email_record.save
        flash.now[:alert] = email_record.errors.full_messages.to_sentence
        @sender_email = SENDER_EMAIL
        @email = email_param
        return render :new, status: :unprocessable_content
      end

      issue_and_deliver_otp(email_record)
      session[:mcp_otp_email_id] = email_record.id
      session[:mcp_otp_sent_at]  = Time.current.to_i
      redirect_to mcp_auth_session_verify_path
    end

    # verify / confirm / resend implemented in Task 5

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

- [ ] **Step 5: Echo the OTP to STDOUT in development**

So the OTP is visible in the `serve-dev` Rails pane during E2E. In `app/models/email.rb`, find `issue_sign_in_otp!` and add a dev-only echo right before it returns `otp`:

```ruby
  def issue_sign_in_otp!
    otp = SecureRandom.alphanumeric(SIGN_IN_OTP_LENGTH)
    update!(
      sign_in_otp_digest: digest_otp(otp),
      sign_in_otp_expires_at: SIGN_IN_OTP_TTL.from_now
    )
    $stdout.puts("[MCP OTP] #{email}: #{otp}") if Rails.env.development?
    otp
  end
```

- [ ] **Step 6: Create the email-entry view**

Create `app/views/mcp/auth_sessions/new.html.erb`:

```erb
<main class="mcp-auth">
  <h1>Connect Read It Soon to Claude</h1>
  <p>Enter your Kindle email address. We'll send a one-time code as a document to your Kindle.</p>
  <p><strong>First:</strong> in your Amazon account, add
     <code><%= @sender_email %></code> to your approved senders list, or the code won't arrive.</p>

  <% if flash[:alert] %><p class="error"><%= flash[:alert] %></p><% end %>

  <%= form_with url: mcp_auth_session_path, method: :post, local: true do |f| %>
    <%= f.email_field :email, value: @email, placeholder: "you@kindle.com", required: true %>
    <%= f.submit "Send code" %>
  <% end %>
</main>
```

- [ ] **Step 7: Run the test, verify pass**

Run: `bundle exec rspec spec/requests/mcp/auth_sessions_spec.rb`
Expected: PASS.

- [ ] **Step 8: Commit**

```bash
git add app/controllers/mcp/auth_sessions_controller.rb app/views/mcp/auth_sessions/new.html.erb app/models/email.rb config/routes.rb spec/requests/mcp/auth_sessions_spec.rb
git commit -m "feat(mcp): OTP auth-session email entry + dev STDOUT OTP echo"
```

---

### Task 5: OTP verify, confirm, and resend (30s gate)

**Files:**
- Modify: `app/controllers/mcp/auth_sessions_controller.rb`
- Create: `app/views/mcp/auth_sessions/verify.html.erb`
- Test: `spec/requests/mcp/auth_sessions_verify_spec.rb`

- [ ] **Step 1: Write the failing tests**

Create `spec/requests/mcp/auth_sessions_verify_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe "MCP auth session verify", type: :request do
  def start_session_with_otp(addr = "reader@kindle.com")
    allow(OtpSignInDeliveryService).to receive(:call)
    post "/mcp/auth_session", params: { email: addr, return_to: "/oauth/authorize?x=1" }
    Email.find_by(email: addr)
  end

  it "rejects an incorrect OTP" do
    start_session_with_otp
    allow_any_instance_of(Email).to receive(:valid_sign_in_otp?).and_return(false)
    post "/mcp/auth_session/verify", params: { otp: "wrong" }
    expect(response).to have_http_status(:unprocessable_content)
    expect(session[:mcp_authenticated_email_id]).to be_nil
  end

  it "accepts a correct OTP and returns to the OAuth flow" do
    email = start_session_with_otp
    allow_any_instance_of(Email).to receive(:valid_sign_in_otp?).and_return(true)
    post "/mcp/auth_session/verify", params: { otp: "anything" }
    expect(response).to redirect_to("/oauth/authorize?x=1")
    expect(session[:mcp_authenticated_email_id]).to eq(email.id)
  end

  it "rejects resend before 30s" do
    start_session_with_otp
    post "/mcp/auth_session/resend"
    expect(response).to have_http_status(:too_many_requests)
  end
end
```

- [ ] **Step 2: Run it, verify it fails**

Run: `bundle exec rspec spec/requests/mcp/auth_sessions_verify_spec.rb`
Expected: FAIL (actions missing).

- [ ] **Step 3: Implement verify/confirm/resend**

In `app/controllers/mcp/auth_sessions_controller.rb`, replace `# verify / confirm / resend implemented in Task 5` with:

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

And add these private helpers below `issue_and_deliver_otp`:

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
Expected: PASS (all three examples).

- [ ] **Step 6: Commit**

```bash
git add app/controllers/mcp/auth_sessions_controller.rb app/views/mcp/auth_sessions/verify.html.erb spec/requests/mcp/auth_sessions_verify_spec.rb
git commit -m "feat(mcp): OTP verify + 30s-gated resend in auth-session flow"
```

---

### Task 6: STDOUT delivery log in `MailgunEmailClient`

**Files:**
- Modify: `app/services/mailgun_email_client.rb`
- Test: `spec/services/mailgun_email_client_spec.rb` (new, minimal)

> Proves at E2E time (in the solid_queue worker pane, which runs `RAILS_LOG_TO_STDOUT=1`) that the EPUB was actually handed to Mailgun.

- [ ] **Step 1: Confirm the existing client shape**

Run: `cat app/services/mailgun_email_client.rb`
Expected: see `MailgunEmailClient.call(to_attributes:, from_attributes:, subject:, body:, attachments: [])`.

- [ ] **Step 2: Write the failing test**

Create `spec/services/mailgun_email_client_spec.rb`:

```ruby
require "rails_helper"
require "tmpdir"

RSpec.describe MailgunEmailClient do
  it "logs a delivery line to STDOUT including the recipient" do
    mailgun_client = instance_double(Mailgun::Client, send_message: true)
    allow(Mailgun::Client).to receive(:new).and_return(mailgun_client)

    dir = Dir.mktmpdir
    epub_path = File.join(dir, "Example Article.epub")
    File.write(epub_path, "epub")

    expect {
      described_class.call(
        to_attributes: [:to, "reader@kindle.com"],
        from_attributes: [SENDER_EMAIL, { first: "Read It", last: "Soon" }],
        subject: "Example Article",
        body: "Body",
        attachments: [{ path: epub_path, name: "Example Article.epub" }]
      )
    }.to output(/\[MailgunEmailClient\] delivering 'Example Article' to reader@kindle\.com \(Example Article\.epub\)/).to_stdout
  ensure
    FileUtils.remove_entry(dir) if dir && Dir.exist?(dir)
  end
end
```

- [ ] **Step 3: Run it, verify it fails**

Run: `bundle exec rspec spec/services/mailgun_email_client_spec.rb`
Expected: FAIL (no STDOUT line).

- [ ] **Step 4: Add the STDOUT log**

In `app/services/mailgun_email_client.rb`, at the start of `self.call`, add:

```ruby
    recipient = Array(to_attributes).last
    attachment_name = attachments.first&.[](:name) || "no attachment"
    $stdout.puts("[MailgunEmailClient] delivering '#{subject}' to #{recipient} (#{attachment_name})")
```

Keep the log before the Mailgun `send_message` call; do not change delivery behavior.

- [ ] **Step 5: Run the test, verify pass**

Run: `bundle exec rspec spec/services/mailgun_email_client_spec.rb`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add app/services/mailgun_email_client.rb spec/services/mailgun_email_client_spec.rb
git commit -m "feat(mcp): log mailgun delivery to STDOUT for E2E verification"
```

---

### Task 7: `SendMarkdownToKindleTool` (MCP::Tool)

**Files:**
- Create: `app/mcp/send_markdown_to_kindle_tool.rb`
- Test: `spec/mcp/send_markdown_to_kindle_tool_spec.rb`

> The tool reads the authenticated `Email` via `server_context[:email_id]`. Title comes from `filename` (basename minus extension). Author: caller value, else email local-part. It reuses the existing pipeline by creating an `Article` and enqueuing `DeliveryJob` (same as `ArticlesController#send_to_kindle`).
> Rails autoloads `app/mcp/send_markdown_to_kindle_tool.rb` as the top-level `SendMarkdownToKindleTool`; no `config/application.rb` change is needed.

- [ ] **Step 1: Write the failing tests**

Create `spec/mcp/send_markdown_to_kindle_tool_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe SendMarkdownToKindleTool do
  include ActiveJob::TestHelper

  def ctx(email) = { email_id: email.id }

  around do |example|
    original_adapter = ActiveJob::Base.queue_adapter
    ActiveJob::Base.queue_adapter = :test
    clear_enqueued_jobs
    example.run
  ensure
    clear_enqueued_jobs
    clear_performed_jobs
    ActiveJob::Base.queue_adapter = original_adapter
  end

  it "enqueues delivery, sets title from filename, passes author" do
    email = create(:email, email: "reader@kindle.com")
    expect {
      resp = described_class.call(
        markdown: "# Hello\n\nBody", filename: "notes/Hello.md", author: "Jane",
        server_context: ctx(email)
      )
      expect(resp).to be_a(MCP::Tool::Response)
      expect(resp.error?).to be_falsey
    }.to have_enqueued_job(DeliveryJob)

    article = email.articles.last
    expect(article.title).to eq("Hello")
    expect(article.author).to eq("Jane")
    expect(article.markdown).to include("Hello")
  end

  it "defaults author to the email local-part when omitted" do
    email = create(:email, email: "reader@kindle.com")
    described_class.call(markdown: "# X", filename: "X.md", server_context: ctx(email))
    expect(email.articles.last.author).to eq("reader")
  end

  it "returns an error response for empty markdown" do
    email = create(:email, email: "reader@kindle.com")
    resp = described_class.call(markdown: "   ", filename: "X.md", server_context: ctx(email))
    expect(resp.error?).to be(true)
  end

  it "returns an error response when the monthly limit is reached" do
    email = create(:email, :with_subscription_period, email: "reader@kindle.com", max_articles_per_month: 1)
    create(:article, email: email, sent_status: :delivered, sent_at: Time.current)

    expect {
      resp = described_class.call(markdown: "# X", filename: "X.md", server_context: ctx(email))
      expect(resp.error?).to be(true)
    }.not_to have_enqueued_job(DeliveryJob)
  end
end
```

`MCP::Tool::Response#error?` is the supported predicate for responses built with `error: true`.

- [ ] **Step 2: Run it, verify it fails**

Run: `bundle exec rspec spec/mcp/send_markdown_to_kindle_tool_spec.rb`
Expected: FAIL (class missing).

- [ ] **Step 3: Implement the tool**

Create `app/mcp/send_markdown_to_kindle_tool.rb`:

```ruby
class SendMarkdownToKindleTool < MCP::Tool
  tool_name "send_markdown_to_kindle"
  title "Send Markdown to Kindle"
  description "Send a Markdown document to the authenticated user's Kindle as an EPUB."
  input_schema(
    properties: {
      markdown: { type: "string", description: "The raw Markdown content to send." },
      filename: { type: "string", description: "Source filename; its basename (no extension) becomes the EPUB title." },
      author:   { type: "string", description: "Author for the EPUB. Infer from the Markdown (frontmatter/byline/title) when present; omit to default to the account name." }
    },
    required: %w[markdown filename]
  )
  annotations(read_only_hint: false, destructive_hint: false, idempotent_hint: false)

  def self.call(markdown:, filename:, author: nil, server_context:)
    email = Email.find_by(id: server_context[:email_id])
    return error("Not authenticated.") unless email
    return error("Markdown is empty.") if markdown.to_s.strip.empty?

    if email.articles_in_period >= email.max_articles_per_month
      return error("Monthly sending limit reached.")
    end

    article = email.articles.create!(
      url: "",
      title: title_from(filename),
      author: author.presence || email.email.split("@").first,
      markdown: markdown,
      sent_status: :scheduled
    )
    DeliveryJob.perform_later(article.id, email.email)

    MCP::Tool::Response.new([{ type: "text", text: "Sending '#{article.title}' to #{email.email}." }])
  end

  def self.title_from(filename)
    base = File.basename(filename.to_s)
    ext = File.extname(base)
    ext.empty? ? base : base.delete_suffix(ext)
  end

  def self.error(message)
    MCP::Tool::Response.new([{ type: "text", text: message }], error: true)
  end
end
```

- [ ] **Step 4: Run the tests, verify pass**

Run: `bundle exec rspec spec/mcp/send_markdown_to_kindle_tool_spec.rb`
Expected: PASS (all four examples).

- [ ] **Step 5: Commit**

```bash
git add app/mcp/send_markdown_to_kindle_tool.rb spec/mcp/send_markdown_to_kindle_tool_spec.rb
git commit -m "feat(mcp): SendMarkdownToKindleTool reusing the delivery pipeline"
```

---

### Task 8: `Mcp::ServerController` — Streamable HTTP MCP endpoint

**Files:**
- Create: `app/controllers/mcp/server_controller.rb`
- Modify: `config/routes.rb`
- Test: `spec/requests/mcp/server_spec.rb`

> Guards `/mcp` with `doorkeeper_authorize!`, builds an `MCP::Server` with the tool, passes `server_context: { email_id: token.resource_owner_id }`, and delegates to a stateless `StreamableHTTPTransport`.

- [ ] **Step 1: Write the failing tests**

Create `spec/requests/mcp/server_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe "Mcp::Server endpoint", type: :request do
  include ActiveJob::TestHelper

  def token_for(email)
    app = Doorkeeper::Application.create!(name: "c", redirect_uri: "http://127.0.0.1/callback", confidential: false)
    Doorkeeper::AccessToken.create!(application: app, resource_owner_id: email.id, expires_in: 3600, scopes: "")
  end

  def jsonrpc(method, params = {}, id: 1)
    { jsonrpc: "2.0", id: id, method: method, params: params }
  end

  let(:headers) { { "Content-Type" => "application/json", "Accept" => "application/json, text/event-stream" } }

  around do |example|
    original_adapter = ActiveJob::Base.queue_adapter
    ActiveJob::Base.queue_adapter = :test
    clear_enqueued_jobs
    example.run
  ensure
    clear_enqueued_jobs
    clear_performed_jobs
    ActiveJob::Base.queue_adapter = original_adapter
  end

  it "401s without a bearer token" do
    post "/mcp", params: jsonrpc("tools/list").to_json, headers: headers
    expect(response).to have_http_status(:unauthorized)
    expect(response.headers["WWW-Authenticate"]).to include(%(resource_metadata="http://www.example.com/.well-known/oauth-protected-resource"))
  end

  it "lists the send tool for an authenticated client" do
    email = create(:email, email: "reader@kindle.com")
    token = token_for(email)
    post "/mcp", params: jsonrpc("tools/list").to_json,
         headers: headers.merge("Authorization" => "Bearer #{token.token}")
    expect(response).to have_http_status(:ok)
    body = JSON.parse(response.body)
    names = body.dig("result", "tools").map { |t| t["name"] }
    expect(names).to include("send_markdown_to_kindle")
  end

  it "calls the tool and enqueues delivery to the token's kindle email" do
    email = create(:email, email: "reader@kindle.com")
    token = token_for(email)
    expect {
      post "/mcp",
           params: jsonrpc("tools/call", { name: "send_markdown_to_kindle",
                                            arguments: { markdown: "# Hi\n\nBody", filename: "Hi.md", author: "Jane" } }).to_json,
           headers: headers.merge("Authorization" => "Bearer #{token.token}")
    }.to have_enqueued_job(DeliveryJob)
    expect(response).to have_http_status(:ok)
    expect(email.articles.last.title).to eq("Hi")
  end
end
```

- [ ] **Step 2: Run it, verify it fails**

Run: `bundle exec rspec spec/requests/mcp/server_spec.rb`
Expected: FAIL (no `/mcp` route).

- [ ] **Step 3: Add the route**

In `config/routes.rb`:

```ruby
match "/mcp", to: "mcp/server#create", via: %i[get post delete]
```

- [ ] **Step 4: Implement the controller**

Create `app/controllers/mcp/server_controller.rb`:

```ruby
module Mcp
  class ServerController < ActionController::API
    include Doorkeeper::Rails::Helpers
    before_action :authorize_mcp!

    def create
      server = MCP::Server.new(
        name: "readitsoon",
        title: "Read It Soon Kindle Sender",
        version: "0.1.0",
        instructions: "Send Markdown documents to the user's Kindle as EPUBs.",
        tools: [SendMarkdownToKindleTool],
        server_context: { email_id: doorkeeper_token&.resource_owner_id }
      )
      transport = MCP::Server::Transports::StreamableHTTPTransport.new(server, stateless: true)
      status, headers, body = transport.handle_request(request)

      response.headers.merge!(headers || {})
      render(json: body&.first, status: status)
    end

    private

    def authorize_mcp!
      doorkeeper_authorize!
    ensure
      attach_resource_metadata_challenge if response.status == 401
    end

    def attach_resource_metadata_challenge
      metadata_url = "#{request.base_url}/.well-known/oauth-protected-resource"
      challenge = response.headers["WWW-Authenticate"].presence || "Bearer"
      return if challenge.include?("resource_metadata=")

      separator = challenge == "Bearer" ? " " : ", "
      response.headers["WWW-Authenticate"] = %(#{challenge}#{separator}resource_metadata="#{metadata_url}")
    end
  end
end
```

The MCP Ruby SDK's Rails controller example uses this contract: `transport.handle_request(request)` returns `[status, headers, body]`, and the Rails response renders `body.first`.

- [ ] **Step 5: Run the tests, verify pass**

Run: `bundle exec rspec spec/requests/mcp/server_spec.rb`
Expected: PASS (all three examples).

- [ ] **Step 6: Commit**

```bash
git add app/controllers/mcp/server_controller.rb config/routes.rb spec/requests/mcp/server_spec.rb
git commit -m "feat(mcp): streamable-HTTP MCP endpoint guarded by doorkeeper"
```

---

### Task 9: Full suite + PR

- [ ] **Step 1: Run the MCP suite**

Run: `bundle exec rspec spec/requests/mcp spec/mcp spec/services/mailgun_email_client_spec.rb`
Expected: all PASS.

- [ ] **Step 2: Run the whole suite for regressions**

Run: `bundle exec rspec`
Expected: green (no new failures vs. baseline).

- [ ] **Step 3: Push + PR**

```bash
git push -u origin mcp-oauth
gh pr create --title "MCP Kindle send + OAuth 2.1 (Rails-native)" \
  --body "AS (Doorkeeper) + DCR + AS/protected-resource metadata + OTP login (30s resend) + MCP Streamable-HTTP endpoint with send_markdown_to_kindle. Spec: docs/superpowers/specs/2026-06-06-mcp-kindle-oauth-design.md"
```

---

## E2E Validation (run by Claude after the plan is built)

> Drives the user's acceptance test against local `http://localhost:4001`. The validator (me) executes these; some steps use the browser (chrome-devtools MCP) and the `claude` CLI.

- [ ] **V1: Start readitsoon**

In `../readitsoon/.worktrees/mcp-oauth` (so the new code is live): run `./serve-dev`. Note the Rails pane port = **4001**. Confirm three panes up: Rails (4001), readability (4002), solid_queue worker (`RAILS_LOG_TO_STDOUT=1`).

- [ ] **V2: Smoke the endpoints**

```bash
curl -s http://localhost:4001/.well-known/oauth-protected-resource | jq .
curl -s -i -X POST http://localhost:4001/mcp -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}' | grep -E 'HTTP/|WWW-Authenticate'
```

Expected: `HTTP/1.1 401 Unauthorized` and a `WWW-Authenticate` challenge containing
`resource_metadata="http://localhost:4001/.well-known/oauth-protected-resource"`.

- [ ] **V3: Verify the test markdown fixture (in readitsoon-mcp repo)**

Verify `fixtures/sample-article.md` already exists and contains:
- H1: `The Quiet Hour Before Dawn`
- Byline: `by Guillermo Siliceo`

- [ ] **V4: Register the MCP server with Claude + complete OAuth (pre-auth)**

```bash
claude mcp add --transport http readitsoon http://localhost:4001/mcp
```

Then trigger interactive auth:
- Run `claude` in the terminal.
- Inside Claude Code, type `/mcp`.
- Select `readitsoon` and choose the authenticate/connect action.

When Claude prints or opens the authorization URL:
- Open it with the browser (chrome-devtools MCP `new_page`/`navigate_page`).
- Enter `guillermo.siliceo@kindle.com`, submit.
- Read the OTP from the **Rails serve-dev pane STDOUT** (`[MCP OTP] guillermo.siliceo@kindle.com: <code>`).
- Type the OTP into the browser, submit; approve consent.
- Confirm Claude reports the server connected/authorized (tokens stored). Verify with `claude mcp list` showing `readitsoon` connected.

- [ ] **V5: Send the file via the headless agent**

```bash
claude -p "Read fixtures/sample-article.md, then use the readitsoon MCP tool send_markdown_to_kindle. Pass markdown as the full file contents, filename as 'The Quiet Hour Before Dawn.md' so the EPUB title matches the H1, and author as 'Guillermo Siliceo' from the byline." \
  --output-format json
```

Confirm the JSON output shows the tool was called and returned:
`Sending 'The Quiet Hour Before Dawn' to guillermo.siliceo@kindle.com.`

- [ ] **V6: Verify delivery actually happened**

In the **solid_queue worker pane** STDOUT, confirm:
`[MailgunEmailClient] delivering 'The Quiet Hour Before Dawn' to guillermo.siliceo@kindle.com (The Quiet Hour Before Dawn.epub)`
This proves the EPUB was handed to Mailgun (Task 6 log). Done = this line appears for the test send.

---

## Self-Review Notes (resolved)

- **Spec coverage:** Rails-only re-scope (whole plan), official ruby-sdk + controller (T7/T8), local token validation via doorkeeper_authorize! (T8), DCR with loopback redirect validation (T3), AS + protected-resource metadata (T2), OTP login + whitelist + dev STDOUT echo (T4), 30s resend (T5), MailgunEmailClient STDOUT log (T6), tool with caller-supplied author + title-from-filename (T7), reuse map (T7), localhost:4001 + serve-dev + claude mcp add + browser OTP + deterministic fixture send (E2E). All covered.
- **Type consistency:** `server_context[:email_id]` set in T8, read in T7. `SendMarkdownToKindleTool` name consistent T7/T8. `title_from`/`error` are T7-private. Tool name string `send_markdown_to_kindle` consistent T7/T8/E2E.
- **Resolved ambiguity pass:** `MCP::Tool::Response#error?` and `StreamableHTTPTransport#handle_request(request)` are now fixed to the current Ruby SDK contract; Task 6 uses the real `MailgunEmailClient.call(...)` signature; Active Job test adapter setup is explicit in specs using `have_enqueued_job`; E2E no longer depends on external `MODEL`/`SYSTEM_PROMPT` values.
