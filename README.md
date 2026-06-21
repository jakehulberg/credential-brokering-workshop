# Give an Agent Real Powers, Safely

You're going to give an autonomous agent the ability to call real APIs, and prove it never holds a
single secret while doing it. The credentials live in a broker; the agent gets capability, not keys.

This repo is your clone-and-go starting point. Everything is copy-paste. You can follow along on your
own machine, or just watch and run it later. Nothing here is graded.

```
   agent (no secrets)  ──HTTPS_PROXY──▶  Agent Vault broker  ──real key──▶  the API
                                              │
                              encrypted creds · per-vault service catalog · audit log · locked egress
```

---

## Prerequisites (do this before the session, ~10 min, needs network)

A quick checklist. Get the **Required** items working at home so the live session is smooth.

**Required**

1. **Agent Vault CLI** (single binary, all platforms):
   ```bash
   curl -fsSL https://get.agent-vault.dev | sh
   agent-vault version
   ```
2. **Docker**, running:
   ```bash
   docker run --rm hello-world
   ```
   macOS: Docker Desktop. Linux: Docker Engine 20.10+. Windows: Docker Desktop works, but the
   container-isolation finale (checkpoint 3) is Linux/macOS only, you'll watch that part. First run of
   the isolation step builds an image (~60s); that's expected.

**Recommended** (the agent-driven steps, the proposals demo and the isolation finale in checkpoint 3, drive a real agent; `curl` covers checkpoints 1-2)

3. **Node.js 18+** (the coding agents below run on it): `node --version`
4. **Claude Code** and a login: `npm i -g @anthropic-ai/claude-code` (checkpoint 3 runs Claude in the cage)
5. **Codex** and an OpenAI/ChatGPT login (see [openai/codex](https://github.com/openai/codex)).

**Optional**

6. A throwaway **GitHub fine-grained PAT** and/or **Stripe test-mode** key to broker a real service.

---

## The three checkpoints

You don't have to keep pace with the room. Each file is self-contained; fall behind and catch up
whenever, or run the whole thing after the stream.

| # | File | What you prove | Needs |
|---|------|----------------|-------|
| 1 | [checkpoints/01-broker-up.md](checkpoints/01-broker-up.md) | A broker running on your machine, owner + vault created | CLI |
| 2 | [checkpoints/02-first-agent.md](checkpoints/02-first-agent.md) | **The aha:** an agent makes an authenticated call holding *no* credential | CLI (+ curl or Claude) |
| 3 | [checkpoints/03-isolation.md](checkpoints/03-isolation.md) | Egress locked at the kernel, the agent *cannot* leave the broker | CLI + Docker (Linux/macOS) |

Quick start:
```bash
git clone https://github.com/jakehulberg/credential-brokering-workshop && cd credential-brokering-workshop
curl -fsSL https://get.agent-vault.dev | sh   # install the Agent Vault CLI (skip if you already have it)
agent-vault server -d       # press Enter at the prompt for passwordless; -d backgrounds it
# then open checkpoints/01-broker-up.md and go
```

---

## What's in here

- `services/`, ready-made service definitions you'll load with `agent-vault vault service add -f`.
- `checkpoints/`, the three guided steps, copy-paste.
- `.env.example`, reference values (the workshop's fake credential plus optional slots for real keys).
  Optional: `source .env` if you'd rather set the env vars than pass `--vault` each time.
- `slides/`, the deck (for reference / re-watching).
- `DESIGN.md`, the Infisical design system the deck (and any other visuals here) follow.

## Mental model (one paragraph)

Agent Vault is a transparent proxy. Your agent gets `HTTPS_PROXY` pointed at it. The broker terminates
TLS with its own CA, matches each request against a per-vault **service** catalog, injects the **real**
credential, and forwards it upstream. The agent's HTTP client needs zero code changes and never sees
the secret. Agents can't edit the catalog themselves, they raise a **proposal** a human approves. And
in container mode, the agent's egress is locked to the broker with iptables, so it physically cannot
reach anything else, cooperative or not.

## After the workshop

- Full walkthrough: the Agent Vault docs **Tutorial**.
- Container isolation deep dive + threat model: docs → Guides → Container isolation.
- Ship your own agent as a container (k8s/Fly/ECS): docs → Guides → Deploy your agent in a container.
