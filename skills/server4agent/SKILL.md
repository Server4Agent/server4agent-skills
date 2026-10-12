---
name: server4agent
description: >-
  Provision and manage persistent agent servers via Server4Agent's MCP tools. Use when asked to create a server or project, run a task on a remote server, deploy to a stable project URL, manage workspace files, or delegate work to an on-server agent.
---

<!--
  GENERATED FILE. Do not edit by hand.

  The tool reference below is rendered from the same table the MCP endpoint
  advertises, so the two cannot drift. To change it, edit lib/mcp/tools.ts (the
  tools) or lib/mcp/skill-doc.ts (the prose and the grouping), then run:

      npm run skill:generate

  tests/skill-doc.test.ts fails when this file is out of date.
-->

Use Server4Agent's MCP tools (the `server4agent` MCP server) to provision
servers, manage projects, and delegate work to an agent that runs on the box.

## One-time setup

Check whether the connection already exists by looking for tools prefixed
`mcp__server4agent__` (for example `mcp__server4agent__create_server`). If they
are there, skip this section.

If they are not, connect the MCP server at `https://mcp.server4agent.com`.
There are two ways in, and the first is better:

**Sign in through the browser (preferred).** Server4Agent runs a standard OAuth
2.1 authorization server with dynamic client registration, so an OAuth-capable
MCP host needs nothing but the URL. Add it as a remote MCP server, and the host
opens a browser to sign in. Nothing to copy, and the token refreshes and can be
revoked from the dashboard.

```bash
your-mcp-host mcp add --transport http server4agent https://mcp.server4agent.com
```

**API key (for hosts without OAuth, and for CI).** Create a key at
https://www.server4agent.com, then pass it as a bearer token:

```bash
your-mcp-host mcp add --transport http server4agent https://mcp.server4agent.com \
  --header "Authorization: Bearer <THEIR_KEY>"
```

For a project config that is checked into git, never embed the literal key. Use
environment variable expansion so each teammate supplies their own:

```json
{
  "mcpServers": {
    "server4agent": {
      "type": "http",
      "url": "https://mcp.server4agent.com",
      "headers": { "Authorization": "Bearer ${SERVER4AGENT_API_KEY}" }
    }
  }
}
```

Each teammate sets `SERVER4AGENT_API_KEY` in their own shell, and the key itself
never gets committed.

## Tools

38 tools. `[read-only]` never writes; `[destructive]` may delete or
overwrite, so confirm with the user before calling one.

### Servers

A server is a sized compute box. Projects live inside it and share its resources.

- `create_server(task?, tier?, kind?, region?, project_sharing?, secrets?)`
  Provision a sized server box. Servers hold projects; each project gets a stable URL that serves once you make it public. Pass `secrets` to set server-wide env vars (shared by every project on the server); values are write-only. Pass `kind: "browser"` for a box with a screen and a browser on it. Returns the server id.
- `list_servers()` `[read-only]`
  List your servers with their tier, kind, status, project sharing, and current project count. kind is standard for a headless box or browser for one with a screen and a browser on it.
- `get_server(server_id)` `[read-only]`
  Get a server's tier, kind, status, capacity, project count, and project sharing. kind is standard for a headless box or browser for one with a screen and a browser on it.
- `update_server(server_id, task?, region?, project_sharing?)` `[destructive]`
  Update a server's nickname, region, or how much its projects may see of each other.
- `control_server(server_id, action)` `[destructive]`
  Pause a server (stop), resume it (start), or reboot it (restart). A paused server stays down until it is started again: its projects are offline, it does not bill, and tools that reach it (prompt, exec, start_app, the file tools, start_build) answer that it is paused instead of waking it.
- `delete_server(server_id)` `[destructive]`
  Tear down a server and every project on it.
- `get_usage()` `[read-only]`
  Get the account's AI spend and budget: how much has been spent this billing period, the plan's included credits, any budget cap, and whether new tasks are blocked (with the reason). Call this before a long series of prompts to check remaining budget, since dispatch fails once a spend ceiling is reached.

### Projects

A project is a workspace plus an app inside a server, with its own slug and URL. New projects are private: the URL only serves once you make it public.

- `list_templates()` `[read-only]`
  List the available project starters. Each template has an id, a human label, a one-line description, default visibility/lifecycle, and a source_kind. The first entries are import templates (source_kind "git" or "archive") that seed a project from your own code; the rest are ordinary starters (source_kind null). Pass the id as `template` to create_project to pre-pick sensible defaults for that use case.
- `create_project(server_id, name?, description?, visibility?, access?, access_password?, lifecycle?, template?, source?, secrets?)`
  Create a project on a server. Projects default to private; pass visibility="public" to expose <slug>.apps.server4agent.com. Use lifecycle="ephemeral" for one-shot work that should be cleaned up when done (call cleanup_project after); default is "persistent" for long-lived monitors / dashboards / tools. Pass `template` (see list_templates) to apply a starter's defaults, or a `source` to seed the workspace from your own code (kind git/archive); name is derived from the repo/archive when omitted. Pass `secrets` to set project-scoped env vars at creation (values are write-only). Explicit fields override the template. Fails if the server is at its tier's project cap. Returns `listen_port`, the port inside the box that this project's URL is routed to: a project serves only while a process is listening there, so bind that rather than a port of your own choosing.
- `list_projects(server_id?)` `[read-only]`
  List projects. Optionally filter by server_id to see what's already on a particular server before creating a new one.
- `get_project(project_id)` `[read-only]`
  Get a single project: name, slug, description, public URL, listen port, run plan, timestamps. listen_port is the port inside the box that this project's URL is routed to, so an app here binds that rather than a port of its own choosing, and has to be running for the URL to answer. listen_port is null on a project created before the field existed, and on a box old enough to predate the in-pod proxy the published port (8080) is reached directly instead, so treat those two as the fallback when an app that binds listen_port still cannot be reached. url_status covers only whether the address and its certificate are ready, so it reads "active" for a public project with nothing listening; it is not a health check. run_plan is how bring-up installs and starts it, or null if no plan has been settled yet; a run_plan whose port is null means the project isn't a server.
- `update_project(project_id, name?, description?, visibility?, access?, access_password?, lifecycle?, run_plan?)` `[destructive]`
  Rename a project, update its description, change visibility, link access or lifecycle, or correct its run plan. The slug + URL stay the same. Setting run_plan only records how the project should be brought up; call init_project when you want it rebuilt.
- `init_project(project_id, proceed?)`
  Start or resume bring-up for a project seeded from a git/archive source: the on-server agent installs dependencies, starts the app, and confirms it responds. Returns a run_id to poll with get_run. Pass proceed=true to continue past still-missing optional secrets. Not for a project whose files you wrote yourself: start that one with start_app.
- `export_project(project_id, ttl_seconds?)`
  Package a project into a portable bundle and return a time-limited download link. The bundle carries the project's source tree, its settings, its run plan, and the NAMES of the env vars it needs. Left out: dependency caches, git history, and any .env or credential file in the workspace, so the bundle never carries a secret value. Hand the file to import_project on this deployment or any other Server4Agent deployment to recreate the project there.
- `import_project(server_id, bundle, name?)`
  Recreate a project on a server from a bundle that export_project produced, here or on another Server4Agent deployment. Pass the bundle base64 encoded in `bundle`; large bundles are better uploaded from the dashboard, since the request body has a size limit. The new project is always private, gets its own slug and URL, and starts bringing itself up right away (the bundle only reaches the server as part of that run). Env vars the bundle names are bound only where this account already holds a value, and no .env file travels: the result lists what is still missing in `missing_env`, and the app will not run until you set those with set_secret.
- `cleanup_project(project_id)` `[destructive]`
  Clean up an ephemeral project once its task is done. Deletes the workspace, retires the slug, and frees the project slot. Errors on persistent projects; use delete_project for those.
- `delete_project(project_id)` `[destructive]`
  Delete a project. The public URL stops working immediately.

### Secrets

Env vars for servers and projects. Values are write-only: no tool ever reads one back.

- `set_secret(key, value, description?)` `[destructive]`
  Create or update a secret (an env var, key → value) on the account. One secret per key; setting an existing key updates its value. Values are write-only: never returned by any tool. Use attach_secret to bind it to a server or project.
- `list_secrets()` `[read-only]`
  List the account's secrets: keys, notes, and where each is bound (servers/projects). Never returns values; only a masked hint.
- `attach_secret(key, value?, server_id, project_id?)` `[destructive]`
  Bind a secret to a scope so runs there see it as an env var. Server-wide when only server_id is given; project-scoped when project_id is also given. Pass value to set-and-bind in one call; omit it to bind an existing secret by key. A key has one value on the account, shared everywhere it is bound: passing value here changes it for every server and project that key is bound to. Give a project that needs its own value a key of its own.
- `delete_secret(secret_id)` `[destructive]`
  Delete a secret and all of its bindings.

### Agent runtime

Start and keep your app running, delegate work to the agent running on the server, or drive the shell yourself.

- `start_app(project_id, command, name?)` `[destructive]`
  Start a project's app and keep it running: the step that puts a project online after its files are written. Runs `command` in the project folder as a background process with PORT set to the project's routed port (the app must listen on PORT), records it so it comes back after a restart, and waits up to about 10 seconds for the port to answer. Calling it again with the same name restarts it, which is how you pick up changed code. Returns whether the app is answering, its log tail, and the URL. A project's URL serves only while its visibility is public, so set that with update_project when the app should be reachable. For static files, put them under public/ and serve that folder, e.g. "python3 -m http.server $PORT --bind 0.0.0.0 --directory public". Never serve the project folder itself, which holds the project's secrets in .server4agent/env.
- `prompt(server_id, prompt, project_id?, follow_up_of?)`
  Send a freeform prompt to the on-server agent. It plans, writes code, runs commands, and ships whatever the prompt asks for. Returns a run_id to poll with get_run. Each call starts a new conversation unless you pass follow_up_of: the run_id of an earlier run to continue, so the agent remembers what it did there. Use it to answer a run that is awaiting_input, or to ask for changes to what a run built. Pass project_id to work inside one project's workspace, or omit it to work across the whole server workspace (e.g. scaffolding a new project).
- `exec(server_id, command, project_id?)` `[destructive]`
  Run a shell command on the server and return its output. Pass project_id to run inside that project's workspace folder, with PORT set to the project's routed port; omit it to run at the server workspace root. For one-off work: installing dependencies, building, inspecting files. Runs as the workspace user, the same one the agent and the apps run as, so what it creates stays editable by both; prefix a command with sudo when it needs root. A project command also gets the project's secrets as env. The command's session ends with the call, so do not use it to start an app or any other long-running process: use start_app, which keeps it running, brings it back after a restart, and tells you whether it answered. Output comes back for about 20 seconds: a command that runs longer keeps going on the server, but its output does not return, and the server's other commands wait for it. Start long work (a big install, a long build) in the background with its output in a file, then read the file.
- `get_run(server_id, run_id)` `[read-only]`
  Get a run's status, its latest progress note, the agent's final response, and what it changed in the workspace (commit, branch, files). Poll this with the run_id returned by prompt or init_project.
- `get_logs(server_id, project_id?, lines?)` `[read-only]`
  Read the latest log tails from a server: one entry per log file that a project's processes have written (each long-running process appends to .server4agent/<name>.log in its project folder). Read live from the server at the moment you call. If the server does not answer in time (starting, restarting, or paused), you get the tails saved earlier with stale: true and a note saying how old they are. An app keeps the same log file across restarts. Pass project_id to get only that project's logs. Use this to debug a crashing app or check what a background process is doing.

### File workspace

Read and write the project's files directly when you do not need the agent.

- `list_files(server_id, project_id?)` `[read-only]`
  List paths in the server's workspace. Pass project_id to list only that project's folder (paths come back relative to the project); omit it to list the whole server workspace.
- `read_file(server_id, path, project_id?, encoding?)` `[read-only]`
  Read a file from the server's workspace. With project_id, path is resolved inside that project's folder; without it, path is relative to the server workspace root. Use encoding:"base64" for binary files (e.g. .xlsx, images); plain text mode will corrupt non-text bytes.
- `write_file(server_id, path, content, project_id?, encoding?)` `[destructive]`
  Write or overwrite a file in the server's workspace. With project_id, path is resolved inside that project's folder; without it, path is relative to the server workspace root. Use encoding:"base64" to write binary content (e.g. a generated .xlsx); plain text mode assumes UTF-8 and will corrupt binary bytes. One call carries up to 512 KB of content; write anything larger in pieces, or fetch it on the server with exec. Writing a file does not serve it: a project's URL is answered by a running app, so after writing the files call start_app (a changed server needs start_app again to pick the change up).
- `delete_file(server_id, path, project_id?)` `[destructive]`
  Delete a file from the server's workspace. With project_id, path is resolved inside that project's folder; without it, path is relative to the server workspace root.
- `get_download_url(server_id, path, project_id?, ttl_seconds?)`
  Mint a time-limited link to download one file from the server's workspace, e.g. after generating a report or converting a file, hand this URL back to the user instead of the raw content. With project_id, path is resolved inside that project's folder. The link needs no login and works for anyone who has it, so only mint it for files that are OK to share.

### Build and deploy

An agent run that builds from a goal, and deploy, which puts a running project online.

- `start_build(server_id, goal, project_id?)`
  Start an agent run on the server that builds something from a goal, working through each stage in order: plan, scaffold, write the code, install, build and test, then start it on the project's port. Returns a build_id to poll with get_build, and the run_id of the run doing the work. Pass project_id to build inside that project's workspace folder; omit it to build at the server workspace root. Counts against the AI budget like prompt.
- `get_build(server_id, build_id)` `[read-only]`
  Get a build's status. The steps move together: running while the run works, done when it finished, failed when it did not (status still reads completed, and error says why). url is the project's address once it is built and public. run_id is the run doing the work: get_run shows its live progress, and when run_status is awaiting_input, answer its question with prompt and follow_up_of set to that run_id.
- `deploy(server_id, project_id?)`
  Put a project on the internet. Checks that something in the project is listening on its `listen_port`, makes the project public, and checks its URL answers, then returns that URL with status live (or publishing, when it is public but has not answered yet). It builds and starts nothing: when nothing is listening it refuses and leaves the project as it was, so start the app first with start_app, or have the on-server agent build and start it with prompt. Pass project_id; a server with exactly one project deploys that one without it.

### Self-tuning

Feed an evaluation report back into the agent's own code so it can fix what the report found.

- `start_tuning(project_id?, server_id?, proposal_id, evaluation_api_base?, key_secret?)`
  Fix an agent from an evaluation platform's report. Fetches the diagnosis for an improvement proposal (the failing test cases and why each answer was marked down), writes it into the project's workspace, and starts an agent run that fixes the cause, redeploys, and submits a fresh evaluation so the result can be compared against the original. Pass project_id when the agent's code is already a project here. Pass server_id instead when it is not: the agent's repository (or archive) is fetched from the evaluation platform and built into a new project on that server, and the fix starts once it is up. The platform names the environment variables the agent needs; their values come from this account's secrets, so set those first with set_secret. Requires a secret holding the evaluation API key (AGENTX_API_KEY unless you name another). Follow it with get_tuning_session.
- `get_tuning_session(tuning_session_id)`
  Get a tuning run's status and the agent run behind it. Reading is what advances it: the first read after the run finishes reports the outcome back to the evaluation platform. Poll this until status is reported or failed.
- `list_tuning_sessions(project_id?, status?)` `[read-only]`
  List tuning runs on this account, newest first. Optionally filter by project.
- `report_tuning_result(tuning_session_id, change_summary?, changed_files?, deployment_url?, failed?)`
  Tell the evaluation platform what a tuning run changed, and where the fixed build is reachable. Use this when you want the summary a reviewer reads next to the before-and-after scores to be your own account rather than whatever the run's final message happened to say.

## Common workflows

**Provision and deploy a project**
1. `create_server` returns a `server_id`.
2. `create_project(server_id, name, template?, visibility="public")` returns a
   `project_id` and the `listen_port` that project's URL is routed to.
3. `prompt(server_id, "Build ...", project_id)` and the on-server agent writes the
   code, binds that port, and leaves the process running. To write the code
   yourself instead, see the next workflow.
4. `deploy(server_id, project_id)` once the run finishes. It checks the app is
   listening, makes the project public if it is not, and returns the URL once
   it answers.

**Put an app you write yourself online**
Nothing here serves a directory. A project's URL is routed to one port inside
the box, so a page or a server is only live once a process is answering on
that port. `start_app` is that step, and the only one you need for it:

1. `create_project(server_id, name, visibility="public")` returns `project_id`
   and `listen_port`.
2. `write_file(server_id, "public/index.html", "<html>...", project_id)`, and
   the same for each other file. Static pages go under `public/`: the
   project folder itself holds
   `.server4agent/env`, the secrets bound to this project, and a file server
   rooted at the project would publish them.
3. `start_app(project_id, command)`. It runs the command in the background
   with `PORT` set to the project's port, records it so the app comes back
   after a restart, and waits for the port to answer.

```
start_app  project_id, command: "node server.js"
start_app  project_id, command: "python3 -m http.server $PORT --bind 0.0.0.0 --directory public"
```

4. The result says whether the app is answering, with its log tail and the
   URL. If it exited, the log says why: fix that and call `start_app` again.
   Calling it again with the same `name` is also how a changed server picks
   up new code.

Your app has to listen on `$PORT`, not a port of its own. Install
dependencies first with `exec` (`npm install`), which waits for them.
`prompt(server_id, "Build and serve ...", project_id)` is the other path: the
on-server agent writes the code and starts it, for work you would rather not
write yourself.

**Follow up on a run**
Every `prompt` call starts a new conversation. To change what a run built, or
to answer a run whose status is `awaiting_input`, pass its id:
`prompt(server_id, "Add a login page", follow_up_of=run_id)`. The agent picks
up that conversation in the same folder and project, so it knows what it did.

**Ephemeral one-shot task**
1. `create_project(server_id, name, lifecycle="ephemeral")`.
2. `prompt` or `exec` to do the work.
3. `cleanup_project(project_id)` frees the slot when done.

**Check capacity before creating**
`get_server(server_id)` reports projects used against the tier cap. At the cap,
free a slot or move the server up a tier before calling `create_project`,
which otherwise fails.

**Watch spend before a long run**
`get_usage()` reports spend for the period, the budget cap, and whether new
tasks are blocked. Worth a call before a series of `prompt` invocations.

## Notes that save a round trip

- **Some questions are for people, not tools.** The Business or Enterprise
  plan, running in your own cloud, a security questionnaire, or an account
  problem no tool can fix: write to hello@server4agent.com for the person you
  are helping (the form at https://www.server4agent.com/contact ends with a
  human check). A person answers within one business day.
- Projects are **private by default**. A newly created project's `url` is
  `null` until you set `visibility="public"`. The slug and URL do not change
  when you toggle visibility.
- A project URL is answered by **a process, not a folder**. Bind the project's
  `listen_port` (from `create_project` or `get_project`) rather than a port of
  your own: an in-pod proxy owns the one published port and routes each public
  project's hostname to its own `listen_port`. On a box old enough to predate
  that proxy there is no demux and 8080 is published directly, so if the URL
  will not answer on `listen_port`, that is the fallback worth trying.
- **What the URL answers tells you which thing is wrong.** A redirect to a
  "Nothing found" page means nothing is routed to that slug: the project is
  private, or it does not exist (the two are deliberately indistinguishable).
  A 200 reading "server4agent: unknown project" or "server4agent: no project
  routed to this server yet" means the request reached the box but the in-pod
  proxy has no route for it yet, so give reconciliation a moment. A 503
  "Temporarily unavailable" means the routing is right and nothing is
  listening on `listen_port`, which is the one your own app causes. A sign-in
  page means the project's `access` is `team`, so it is serving fine and only
  the account's members get in (an allowlist can narrow it further still). A
  password prompt means `access` is `password`: also serving fine, for
  anyone who has the password. Set or change it with `update_project`
  `access_password`; it is never returned, so ask the owner for it rather
  than looking for it.
- A process started through `exec` **dies with the call**. Start anything
  long-running with `start_app`, which also records it in the project's
  `.server4agent/start.sh` so a restart, a pause and a resume bring it back.
  Other lines in that file are left alone. If you edit it by hand, keep it to
  plain, self-contained commands, one per line: boot and resume do not replay
  it the same way, so a line that leans on a `cd` or an `export` from an
  earlier line works in one and not the other.
- `deploy` **publishes, it does not build**. It refuses when nothing is
  listening on the project's port, so start the app first (`start_app`, or
  `prompt`). On success it returns the project's URL with status `live`,
  or `publishing` when the URL has not answered yet: call it again shortly.
- `create_project` **fails at the tier cap**: small holds 3 projects, medium 10,
  large is unlimited. Moving a server down a tier is refused when the smaller
  cap would not fit the projects already there.
- `prompt` and `start_build` are **asynchronous**. They return an id, and you
  poll `get_run` or `get_build` for progress.
- Secret **values are write-only**. `list_secrets` returns keys, notes, and
  bindings, never a value.
- Argument details beyond the signatures above are in each tool's own JSON
  schema, which your MCP host already has once connected.
