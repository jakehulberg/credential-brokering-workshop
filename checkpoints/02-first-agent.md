# Checkpoint 2: an agent that holds no secrets

**Goal:** make an authenticated API call where the credential is injected by the broker and the agent
(your shell, or Claude) never sees it. You'll watch the injected token appear server-side. ~5 minutes.

We broker against `postman-echo.com`, a public service that reflects your request headers back as
JSON. The broker injects `Authorization: Bearer <WORKSHOP_TOKEN>`; you send nothing, and the response
shows the token materialized.

> **Same place as checkpoint 1:** one terminal on your machine (broker still running in the background
> from `agent-vault server -d`), plus the browser UI for the click-along bits. No containers yet.
> Steps 1 and 2 have a **UI** and a **CLI** way; steps 3 and 4 are CLI (plus the Logs tab in the UI).

## 1. Put the credential in the vault

The value never leaves the broker after this. The agent will never see it.

- **UI:** open the `workshop` vault → **Credentials** tab → **Add credential** → keep the **Static** tab,
  Key `WORKSHOP_TOKEN`, Value `wsk_live_this_is_not_a_real_secret_0000` → **Add**.
- **CLI:** `agent-vault vault credential set WORKSHOP_TOKEN="wsk_live_this_is_not_a_real_secret_0000" --vault workshop`

![Add Credential form](img/add-credential.png)

## 2. Define the service

A **service** says "this host is allowed, and here's how to attach the credential."

**UI:** **Services** tab → **Add service**, then fill the drawer:

1. **Name:** `echo`
2. **Host pattern:** `postman-echo.com`
3. **Enabled:** leave on
4. **Authentication:** click the **Bearer** tab
5. **Token credential:** select `WORKSHOP_TOKEN`
6. Leave **URL substitutions** empty → click **Add service**

**CLI:** `agent-vault vault service add -f services/echo.yaml --vault workshop`
(`services/echo.yaml` defines exactly the same thing.)

![Add Service form](img/add-service.png)

> **Which Authentication type?** The tab you pick decides how the broker attaches the credential:
> - **Bearer** — adds `Authorization: Bearer <credential>`. The common one for APIs (what we use here).
> - **API key** — puts the credential in a header you name (e.g. `X-API-Key`), with an optional prefix.
> - **Basic** — HTTP Basic auth from a username + password credential.
> - **Passthrough** — injects *nothing*; just allowlists the host. The agent sends its own auth, or a
>   placeholder you swap via URL substitutions. Use when the client insists on sending something.
> - **Custom** — arbitrary header templates with `{{ CREDENTIAL }}` placeholders, for odd auth schemes.
>
> Bearer, API key, and Basic = "the agent sends nothing, the broker injects." Passthrough/Custom = "the
> agent sends a placeholder, the broker rewrites it."

> **"How am I supposed to know my upstream's auth scheme? I just have a token."** Mostly you don't have to:
> - **Start from a preset.** Agent Vault ships a catalog of ~22 common APIs (Stripe, GitHub, OpenAI,
>   Anthropic, Jira, Slack, Twilio…). Pick one in the **Add service** drawer and it prefills the auth
>   type, header, and a suggested credential key (`agent-vault catalog` lists them).
> - **Let the broker detect it.** Once an agent has hit a host, it shows up under **"detected in recent
>   traffic"** with the scheme the broker observed, one click to add a service for it.
> - **Genuinely custom?** Use **passthrough** (the agent sends its own auth, the broker just allowlists),
>   or check the API's docs once.

## 3. First, without the broker  (the baseline)

Make the call the ordinary way, no broker in front of it. You send no auth header, so none arrives:

```bash
curl -s https://postman-echo.com/get | grep -i authorization || echo "no auth header, as expected"
```

That's the "before": the request goes out naked. Hold that thought.

## 4. Now run it through the broker

You don't set any environment variables yourself. **`agent-vault run` sets them for the command it
wraps**, `HTTPS_PROXY` pointed at the broker, the CA trust, and your vault scope, so an ordinary HTTP
client transparently routes through the broker. (You're logged in as the owner, so it uses your
session; no token needed for this local run.)

```bash
# plain curl, works for everyone:
agent-vault run --vault workshop -- bash -c 'curl -s https://postman-echo.com/get'

# or the full agent experience with Claude Code:
agent-vault run --vault workshop -- claude
#   then ask: "GET https://postman-echo.com/get with curl and show me the JSON. Send no auth header."

# or with Codex:
agent-vault run --vault workshop -- codex
#   then ask it the same.
```

**Look at the result, same call, only the wrapper changed:**

- **Terminal:** the reflected JSON now shows a header you never sent:
  `"authorization": "Bearer wsk_live_this_is_not_a_real_secret_0000"`.
- **UI:** open the `workshop` vault → **Logs** tab and see the brokered request (method, host, status).

---

✅ **Checkpoint reached when:** the brokered call shows the `authorization` header (and appears in the
Logs tab), and the un-brokered call does not.

Next: [03-isolation.md](03-isolation.md), making it so the agent *cannot* go around the broker.
