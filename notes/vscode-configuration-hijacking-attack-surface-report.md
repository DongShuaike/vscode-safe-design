# VS Code Configuration-Hijacking Attack Surface Report

**Audience:** VS Code maintainers and security researchers  
**Goal:** Identify configuration-driven supply-chain attack vectors similar to `.vscode/tasks.json` `runOn: folderOpen`, and provide practical PoCs + hardening ideas.

---

## 1) Scope and methodology

I reviewed the VS Code core/built-in code paths that translate workspace/repository configuration into executable behavior:

- Task auto-run and task execution logic.
- Task/debug JSON schemas and input-variable command execution.
- Workspace trust gates and trust-bypass settings.
- Extension recommendation and extension activation surfaces.
- Container attach schema hooks that execute commands.

This is focused on **configuration hijacking**, not classic binary exploitation.

---

## 2) Baseline (the attack you described)

Your 3-stage model is accurate:

1. **Poisoning:** attacker plants malicious `.vscode/tasks.json` with `"runOptions": { "runOn": "folderOpen" }`.
2. **Triggering:** user trusts workspace (or uses derivative IDE with weaker trust UX).
3. **Execution/stealth:** task runs in terminal; output visibility can be reduced (`reveal: silent/never`, `echo: false`), and task can be hidden from quick pick (`hide`).

VS Code currently ties folder-open auto-tasks to workspace trust + permission prompt logic, but once users allow auto tasks globally, the risk increases for later repos.

---

## 3) Similar attack vectors (prioritized)

## Vector A — Hidden auto-task chains (`dependsOn`) + stealth presentation

### Why this is similar
Auto task can execute *indirect* payloads via `dependsOn`, and the visible task label can look harmless while child tasks perform the malicious step.

### Relevant mechanics
- `runOn: folderOpen` allows execution on open.
- `dependsOn` and `dependsOrder` can chain arbitrary tasks.
- `presentation.reveal`, `presentation.echo`, and `hide` can reduce visibility.
- OS-specific command branches (`windows`/`osx`/`linux`) allow precise multi-platform payloading in one file.

### Benign PoC
```json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "Project Init",
      "runOptions": { "runOn": "folderOpen" },
      "dependsOn": ["silent-collector"],
      "presentation": { "reveal": "silent", "echo": false }
    },
    {
      "label": "silent-collector",
      "hide": true,
      "type": "shell",
      "command": "echo POC_TASK_CHAIN_TRIGGERED"
    }
  ]
}
```

### Mitigation ideas
- Add **transitive preview** in the trust/allow dialog: show not only the top task, but its `dependsOn` graph.
- New policy setting: block `runOn: folderOpen` when task has hidden dependencies.

---

## Vector B — `launch.json` → `preLaunchTask` execution laundering

### Why this is similar
A repo can move malicious execution from obvious `tasks.json` into `launch.json` so the trigger happens when users press **F5 / Run and Debug**, a common developer action.

### Relevant mechanics
- Debug compounds and configurations can define `preLaunchTask`.
- Debug service executes preLaunchTask before launching the debug target.

### Benign PoC
`.vscode/launch.json`
```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Run API",
      "type": "node",
      "request": "launch",
      "program": "${workspaceFolder}/index.js",
      "preLaunchTask": "quiet-poc"
    }
  ]
}
```

`.vscode/tasks.json`
```json
{
  "version": "2.0.0",
  "tasks": [
    { "label": "quiet-poc", "type": "shell", "command": "echo POC_PRELAUNCH_TASK" }
  ]
}
```

### Mitigation ideas
- Add per-workspace first-run prompt: “This debug config runs task X before debugging.”
- Add warning badge in debug dropdown for configs with `preLaunchTask`.

---

## Vector C — `inputs` of type `command` (indirect command execution in task/debug workflows)

### Why this is similar
`inputs` can execute VS Code commands, which can invoke extension command handlers. This creates an indirection layer that can hide behavior from quick review.

### Relevant mechanics
- `inputs` supports `"type": "command"` + `"command": "..."`.
- Tasks/debug paths that resolve `${input:...}` can invoke these commands.

### Benign PoC
`.vscode/tasks.json`
```json
{
  "version": "2.0.0",
  "inputs": [
    {
      "id": "pickProc",
      "type": "command",
      "command": "workbench.action.terminal.new"
    }
  ],
  "tasks": [
    {
      "label": "input-command-poc",
      "type": "shell",
      "command": "echo ${input:pickProc}"
    }
  ]
}
```

### Mitigation ideas
- Security review mode in UI: render resolved input graph and show command IDs before first run.
- Policy control to block command-type inputs in workspace files unless allowlisted.

---

## Vector D — Workspace extension recommendation pivot (`extensions.json`) → immediate activation events

### Why this is similar
Repo-level `extensions.json` can socially nudge installation of an attacker-controlled extension; once installed, extension code may activate on startup or workspace conditions.

### Relevant mechanics
- Workspace file supports `recommendations`.
- Recommendation notification includes one-click Install.
- Extension activation events include broad triggers such as `onStartupFinished`, `workspaceContains`, and `*`.

### Benign PoC
`.vscode/extensions.json`
```json
{
  "recommendations": [
    "publisher.suspicious-helper"
  ]
}
```

Attack chain concept: malicious extension chooses activation event `onStartupFinished` and runs payload during startup.

### Mitigation ideas
- Improve recommendation dialog to show extension trust metadata (publisher age, install base, signing/reputation signals).
- Add enterprise policy: block workspace-recommended extension install unless publisher is allowlisted.

---

## Vector E — Trust downgrade / trust bypass configurations

### Why this is similar
If users (or distro defaults) disable workspace trust or bypass trust for terminal creation, many configuration-driven protections become much weaker.

### Relevant mechanics
- `security.workspace.trust.enabled` can disable trust globally.
- Terminal setting `terminal.integrated.allowInUntrustedWorkspace` explicitly bypasses a trust protection, with security warning in description.

### Benign PoC
User settings:
```json
{
  "security.workspace.trust.enabled": false,
  "terminal.integrated.allowInUntrustedWorkspace": true
}
```

### Mitigation ideas
- High-visibility security banner whenever these settings are active.
- Require explicit one-time re-confirmation after upgrade or profile import.

---

## Vector F — Container attach configuration hooks (`postAttachCommand`, auto extension install)

### Why this is similar
Container-related config can run commands post-attach and request extension installs in container context. This can be embedded in project metadata and triggered in common “open in container” workflows.

### Relevant mechanics
- `attachContainer` schema includes `extensions` list (install into container).
- `postAttachCommand` runs shell/command after attach.

### Benign PoC
`attachContainer.json`
```json
{
  "workspaceFolder": "/workspace/project",
  "extensions": ["ms-python.python"],
  "postAttachCommand": "echo POC_POST_ATTACH"
}
```

### Mitigation ideas
- Show command diff preview before first attach and require explicit approval.
- Policy gate: disallow `postAttachCommand` in untrusted repos.

---

## 4) Observations on current VS Code defenses

- Auto tasks are already guarded by workspace trust and an allow prompt, with default `task.allowAutomaticTasks = off`.
- Terminal creation is blocked in untrusted workspaces by default, but there is an explicit bypass setting.
- The protection posture is strongest when users keep trust enabled and avoid global “always allow” decisions.

---

## 5) Recommendations for the VS Code community

1. **Threat-model config files as executable policy** (`tasks.json`, `launch.json`, `extensions.json`, container attach/devcontainer metadata).
2. **Introduce “execution provenance UI”**: a single security panel that explains “what this workspace can auto-run and through which chain.”
3. **Add transitive graph inspection** for tasks/debug (`runOn`, `dependsOn`, `preLaunchTask`, `${input:*}` command nodes).
4. **Enterprise controls**:
   - Allowlist extension publishers for workspace recommendations.
   - Block command-type inputs from workspace files by policy.
   - Block folder-open tasks by policy unless signed/approved.
5. **Improve silent-behavior detection**:
   - Flag tasks that combine `runOn: folderOpen` + `reveal: never/silent` + `echo:false` + hidden dependencies.

---

## 6) Quick triage checklist for defenders

- Inspect `.vscode/tasks.json`: `runOn`, `dependsOn`, `hide`, `presentation.reveal`, `echo`.
- Inspect `.vscode/launch.json`: `preLaunchTask` / `postDebugTask`.
- Inspect task/debug `inputs`: `type: command` and command IDs.
- Inspect `.vscode/extensions.json`: unfamiliar publishers.
- Inspect container attach/devcontainer hooks: `postAttachCommand`, extension auto-installs.
- Verify user/org policy: workspace trust enabled, automatic tasks off-by-default, no untrusted-terminal bypass.

---

## 7) Responsible PoC use

All PoCs above are harmless (`echo`-style) and intended for validation/training only. Do not execute unreviewed workspace configuration from untrusted repositories.
