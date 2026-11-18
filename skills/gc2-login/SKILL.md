---
name: gc2-login
description: Use when a GC2 MCP tool answers "Not logged in to GC2" or "session expired", when the user wants to log in, log out, switch GC2 host/database/user, or asks which GC2 connection is active — the gc2 CLI session the MCP server runs on (gc2 whoami, gc2 connect, gc2 login --flow device|code, gc2 logout), and how an agent starts a login flow on the user's behalf.
---

# GC2 Login (CLI session for the MCP tools)

The GC2 MCP server has no login of its own. It uses the session of the `gc2` CLI (`@mapcentia/gc2-cli`), stored in `~/.config/configstore/gc2-env.json`: host, user, database, access token and refresh token. Logging in with the CLI is logging in for the MCP tools.

This skill is about that operator session. Auth inside application code is `gc2-auth-model`.

## Hard rules

- **Never start a login, change the connection or log out without asking the user first.**
- **Always show the current connection** (`gc2 whoami`) before a login and ask whether it should be kept or changed.
- **Only two flows may be offered: `device` and `code`.** Never suggest, mention as an option, or run the password flow (`--flow password`, `-u`, `-p`), and never ask the user for a password.
- Never read, print or copy tokens from the config file.

## 1) Check the session

```bash
gc2 whoami --json
# {"host":"https://api.gc2.io","user":"jane","database":"mydb","superUser":true,"status":"active"}
```

| `status` | Meaning | Action |
|---|---|---|
| `active` | Access token valid | Nothing; report the connection if asked |
| `expired` | Access token expired — the MCP server normally renews it with the refresh token on the next call | Try the tool call first; log in only if it answers "session expired" |
| `none` | Not logged in | Go to step 2 |

An empty `host` means the default, `https://api.gc2.io`. If `gc2` is not installed, ask before installing it: `npm i -g @mapcentia/gc2-cli`.

If `API_TOKEN` (and optionally `API_BASE_URL`) is set in the MCP server's environment, the server uses that and ignores the CLI session — a login will not change what the tools do; say so instead of starting a flow.

## 2) Ask the user

When the user is not logged in (or a tool reported an expired session), stop and ask — in one message:

1. State the connection from `whoami`: host, and user/database if set.
2. Ask whether to log in **on that host or change it** to another.
3. Ask whether you should start the login flow, and with which flow: **device (recommended)** or code.

Proceed only on a yes.

## 3) Change the connection (only if asked)

```bash
gc2 connect https://my-gc2-host.example   # always pass the host; without it the command prompts
```

The host must include the scheme. `connect` checks `<host>/ping`, then clears user and database — **a login must follow**, and `whoami` can still read `active` from the old host's token until it does. After a host change, restart the MCP server (reload plugins / restart the client) so it talks to the new host.

`gc2 logout` clears the session but keeps the host; `gc2 connect --reset` clears everything, host included.

## 4) Log in

### Device flow — recommended

Works from any shell: nothing has to open a browser on this machine or redirect back to it. The user approves on any device.

`gc2 login` waits for a key press before it starts polling, so an agent shell must feed it a newline. With closed stdin it exits 0 **without logging in**.

1. Start it **in the background**, capturing stderr (all output goes there):
   ```bash
   echo | gc2 login --flow device 2> /tmp/gc2-login.log
   ```
2. Read the log and give the user the two lines, verbatim:
   ```
   Your code  XXXX-XXXX
   Open       https://<host>/device
   ```
3. Wait for the command to finish — it polls until the user has approved (or the code expires). Do not run it in the foreground: you would not see the code until it times out.
4. Success ends with `✔ Signed in as <user>` and `Database <db>`. A failure prints `✖ <reason>` and exits 1 — report it and offer to start again.

If you cannot run background commands, ask the user to run `gc2 login --flow device` in their own terminal (in Claude Code: `! gc2 login --flow device`).

### Code flow

Authorization code with PKCE: the CLI opens the system browser and receives the redirect on `http://127.0.0.1:5657/auth/callback`. It therefore needs a browser **on the same machine** and a free port 5657 — it does not work over SSH, in containers or in remote/cloud sessions. Use it only when the user picks it; it is what a bare `gc2 login` runs.

Best run by the user in their own terminal: `gc2 login`. From an agent shell, `echo | gc2 login --flow code 2> /tmp/gc2-login.log` in the background opens the browser and waits for the redirect.

## 5) Verify and report

```bash
gc2 whoami
```

Report host, user, database and whether the user is an admin (super user). A new login on the same host is picked up by the MCP server on the next tool call — retry the call that failed. Sub-user logins get `403` on super-user-only tools; that is a privilege matter (`gc2-privileges`), not a login failure.

## Common mistakes

| Mistake | Reality |
|---|---|
| Starting a login as soon as a tool says "Not logged in" | Ask first; show the connection and let the user keep or change it |
| Offering the password flow, or asking for credentials | Only `device` or `code`; the agent never handles a password |
| Running `gc2 login` in the foreground of an agent shell | It blocks; the device code is not visible until timeout. Background it and read the log |
| Running it without piping a newline | Exits 0 silently, no login |
| Treating `status: expired` as logged out | The refresh token usually still works; try the tool first |
| `gc2 connect` without a host argument | It prompts and hangs in an agent shell |
| Changing host and expecting the tools to follow at once | Log in again, then restart the MCP server |
| Logging in while `API_TOKEN` is set for the server | The env token wins; the CLI session is not used |
