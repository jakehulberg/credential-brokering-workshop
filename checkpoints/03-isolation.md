# Checkpoint 3: Locked egress, the agent *cannot* leave the broker

**Goal:** prove the guarantee isn't cooperative. In container mode the agent's network egress is
locked with iptables, so it can reach the broker and nothing else, no matter what it tries. ~7 minutes.

> **Linux / macOS only.** Container isolation errors on Windows in v1. Windows users: watch this one;
> everything you proved in checkpoint 2 already holds in host mode.
> **Needs Docker running.** **Two terminals:** terminal A runs the agent in the cage; terminal B
> execs into its container to inspect the egress lock. The broker keeps running in the background.

## 1. Run the agent inside an isolated container  (terminal A)

Run this from a **project directory**, not your home folder. The container mounts your current
directory as its `/workspace`, and it refuses to mount a directory that contains `~/.agent-vault`
(the broker's encrypted data), so running from `~` errors out.

```bash
cd ~/credential-brokering-workshop      # or any folder that isn't your home dir
agent-vault vault run --isolation=container --share-agent-dir --vault workshop -- claude
```

`--share-agent-dir` reuses your **existing host Claude login** inside the container (on macOS it bridges
your Keychain credential in). Without it, Claude tries to log in *inside* the sandbox and the OAuth
flow can't complete through the locked egress. First run builds the isolation image (~60s).

Claude now runs inside a Docker container whose only allowed TCP destination is the Agent Vault proxy.
The brokered call still works just like checkpoint 2:
> "GET https://postman-echo.com/get with curl and show me the JSON."

(Watch it land in the vault **Logs** tab in the UI, same as before.)

## 2. Prove the lock  (terminal B)

While Claude is running, exec into its container:

```bash
docker ps                                   # find the agent-vault isolation container
docker exec -it <container-id> sh
```

Inside the container:

```bash
iptables -S OUTPUT                              # default policy: DROP

# THROUGH the broker (HTTPS_PROXY is set in here): works
curl -m 5 https://postman-echo.com/get

# AROUND the broker (--noproxy forces a direct connection): hangs, then killed by the kernel
curl -m 5 --noproxy '*' https://example.com
```

Going *through* the broker is the only path out; the direct connection is dropped at the kernel before
it leaves the container. There's **no DNS rule** either, so the agent can't even exfiltrate over a
lookup. It could unset every env var, spawn a subprocess, open a raw socket: the only way out is the
broker.

> **Want the broker to be a strict allowlist too?** Flip the vault's unmatched-host policy to **deny**
> (vault → **Settings** in the UI; default is passthrough). Then even *through* the broker, only hosts
> with a service get out, everything else gets a `403`. In deny mode you must allowlist whatever the
> agent itself calls (e.g. `api.anthropic.com` for Claude), which is why it fits locked-down/production
> agents better than a live interactive Claude session.

## 3. Why this matters

Host mode (checkpoint 2) is cooperative: it trusts the agent to honor `HTTPS_PROXY`. A malicious or
prompt-injected agent could try to bypass it. Container mode removes the cooperation: egress is
**enforced** at the kernel. Going *around* the broker is impossible; going *through* it is the only
path, and (in deny mode) what the broker forwards is governed by your service catalog.

---

✅ **Checkpoint reached when:** `iptables -S OUTPUT` shows `DROP`, the direct (`--noproxy`)
`example.com` call hangs/fails, and the proxied `postman-echo.com` call succeeds.

That's the whole story: capability without credentials, and egress you control at the kernel. See the
repo README for where to go next.
