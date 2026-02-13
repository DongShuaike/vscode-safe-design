# Jamf VS Code attack mechanism summary

This summary is aligned with the configuration-hijacking model you provided.

## Core mechanism

1. **Poisoning phase**
   - Attacker places a malicious `.vscode/tasks.json` in a repository.
   - Uses `"runOptions": { "runOn": "folderOpen" }` to auto-trigger on folder open.

2. **Trigger phase**
   - Attack depends on trust UX decisions:
     - In VS Code, this intersects with Workspace Trust and task-allow prompts.
     - In derivative IDEs, weakened trust prompts can turn this into near-silent execution.

3. **Execution phase (stealth + reach)**
   - Command runs in integrated terminal.
   - Stealth options include `presentation.reveal: "silent"` and hidden task UX patterns.
   - Cross-platform branches let one config target Windows/macOS/Linux.

## Key risk framing

This is not a traditional memory exploit; it is abuse of intended automation features in a trusted developer workflow.

## Extended research

For a broader analysis of adjacent vectors (debug `preLaunchTask`, command inputs, extension recommendation pivoting, trust bypass settings, and container attach hooks), see:

- `notes/vscode-configuration-hijacking-attack-surface-report.md`
