  D1 — The zip

  Path on this box:
  /home/root00x/Downloads/pai-copilot-bundle.zip

  Stats: 65 KB · 64 files · SHA-256 6aa9d137cd3814dd3fb4a909862a2f3bb7b2eaf58b6f898556bbace29555a532

  Top-level entries (when unzipped): .github/, .vscode/, MEMORY/, probe/, README.md, OPEN_QUESTIONS.md, .gitignore — these mirror your work repo's root.

  Get it onto your Windows machine (pick whichever applies):
  - WSL: it's already on your Linux side — copy from /home/root00x/Downloads/pai-copilot-bundle.zip to /mnt/c/Users/<you>/Downloads/
  - SCP: scp root00x@<this-host>:~/Downloads/pai-copilot-bundle.zip C:\Users\<you>\Downloads\
  - Shared drive: drag/drop via your file manager

  D2 — VSCode install walkthrough

  ▎ Run all PowerShell commands from your work repo's root directory unless otherwise noted. Replace <repo> with your actual repo path.

  Phase 1 — Drop the bundle in (10 min)

  Step 1. Extract the zip:
  cd "$env:USERPROFILE\Downloads"
  Expand-Archive -Path .\pai-copilot-bundle.zip -DestinationPath .\pai-copilot-bundle -Force
  
  Step 2. Open your work repo in VSCode:
  code <repo>
  
  Step 3. From a PowerShell terminal at the repo root, copy bundle contents:
  $src = "$env:USERPROFILE\Downloads\pai-copilot-bundle"
  xcopy /E /I /Y "$src\.github"  ".github"
  xcopy /E /I /Y "$src\.vscode" ".vscode"
  xcopy /E /I /Y "$src\MEMORY"  "MEMORY"
  xcopy /E /I /Y "$src\probe"   "probe"
  Copy-Item "$src\README.md"         ".\PAI_README.md"
  Copy-Item "$src\OPEN_QUESTIONS.md" ".\PAI_OPEN_QUESTIONS.md"
  Get-Content "$src\.gitignore" | Add-Content .gitignore

  ▎ The README and OPEN_QUESTIONS files are renamed PAI_*.md so they don't collide with your repo's existing README.

  Step 4. Verify Python launcher:
  py -3 --version
  If py isn't found: install Python 3.10+ from python.org (the installer adds the launcher), or edit .github\hooks\hooks.json and .vscode\mcp.json — change every "command": "py" to "command": 
  "python" and remove the "-3" from args.
  
  Step 5. Reload VSCode so Copilot picks up the new files:
  Ctrl+Shift+P  →  "Developer: Reload Window"

  Step 6. First smoke test — type into Copilot Chat:
  @principal hello
  Expected: a terse first-person reply with the structure INTENT / ROUTE / ISA / NEXT. If you get a generic reply, the agent file isn't loading — confirm .github/agents/principal.agent.md
  exists and reload again.

  Step 7. Second smoke test:
  /algo build a quick test thing
  Expected: the agent walks the OBSERVE phase and creates MEMORY/WORK/build-a-quick-test-thing/ISA.md (or similar slug) with frontmatter phase: OBSERVE. Open the file to confirm.

  Phase 0 — Verify hook contract BEFORE relying on Phase 2 (10-15 min)

  ▎ Do this BEFORE assuming the security and ISA-sync hooks actually fire. The Copilot Preview hook contract is documented but not yet rock-stable.

  Step 8. Copy the probe into .github/scripts/:
  Copy-Item .\probe\probe_hook.py .github\scripts\probe_hook.py -Force

  Step 9. Open .github/hooks/hooks.json and TEMPORARILY add a probe entry under each event. Easiest way: replace the file with this version, then revert after Step 11:
  {
    "hooks": {
      "SessionStart":     [{ "command": "py", "args": ["-3", ".github/scripts/probe_hook.py"] }],
      "UserPromptSubmit": [{ "command": "py", "args": ["-3", ".github/scripts/probe_hook.py"] }],
      "PreToolUse":  [{ "matcher": ["*"], "command": "py", "args": ["-3", ".github/scripts/probe_hook.py"] }],
      "PostToolUse": [{ "matcher": ["*"], "command": "py", "args": ["-3", ".github/scripts/probe_hook.py"] }],
      "Stop":        [{ "command": "py", "args": ["-3", ".github/scripts/probe_hook.py"] }]
    }
  } 
  
  Step 10. Reload VSCode again. Run 5 actions in Copilot Chat: read a file, edit a file, run a terminal command, send any chat message, close+reopen the chat.

  Step 11. Read the captured log:
  Get-Content .\probe\probe-log.jsonl

  Confirm that each entry has parsed_stdin populated with tool_name and tool_input keys. If structure differs from PAI_OPEN_QUESTIONS.md Item 1 — message me back with one or two log lines and
  I'll adjust the scripts.

  Step 12. Restore the real hooks:
  git checkout .github\hooks\hooks.json
  Remove-Item .github\scripts\probe_hook.py
  (Or revert manually if not yet committed.)

  Phase 2 — Confirm hooks fire (5 min)

  Step 13. Test ISA-sync — edit any ISA frontmatter phase: field:
  notepad .\MEMORY\WORK\example\ISA.md
  # change phase: COMPLETE → phase: VERIFY, save, close
  Get-Content .\MEMORY\WORK\work.json
  Expected: work.json contains an example session entry with phase: VERIFY.

  Step 14. Test security pipeline — try a deny-listed command in the integrated terminal:
  git push --force origin main
  Expected: [PAI SECURITY] BLOCKED: destructive force-push and the command doesn't run. (If it runs, the hook contract assumption is wrong — go back to Phase 0.)

  Phase 3 — Databricks MCP (15 min)

  Step 15. Install MCP runtime:
  py -3 -m pip install -r .vscode\mcp-servers\databricks_mcp.requirements.txt

  Step 16. Confirm Databricks CLI is authenticated:
  databricks auth describe
  databricks catalogs list --output json | Select-Object -First 5
  If either fails: databricks auth login and complete the browser flow before continuing.

  Step 17. Reload VSCode. Test the MCP:
  @data-analyst list catalogs
  Expected: the agent invokes databricks/list_catalogs and quotes the JSON output. If it can't find the tool, check Settings > Copilot > MCP — your org may have MCP servers disabled (see
  PAI_OPEN_QUESTIONS.md item 10).

  Phase 4 — Knowledge (5 min)

  Step 18. Try the knowledge add flow:
  /knowledge-add
  Walk through the 5-question interview. Confirm a new file appears at MEMORY/KNOWLEDGE/<type>/<slug>.md with related: populated.

  Dynamic agent creation (verify the headline feature)

  Step 19. Create your first custom agent:
  /create-pai-agent
  Pick a real recurring task you do at work — e.g., name notebook-pr-reviewer, purpose "reviews Databricks notebook PRs for SQL safety and lineage." Walk the 5 questions.

  Step 20. Confirm the agent is invocable:
  @notebook-pr-reviewer review this notebook
  And confirm .github/agents/principal.agent.md handoffs: list now contains notebook-pr-reviewer.

  ━━━ ✅ VERIFY ━━━ 6/7

  ✅ Zip exists: /home/root00x/Downloads/pai-copilot-bundle.zip (65K, 64 entries, SHA-256 above).
  ✅ Install guide covers all 4 phases + Phase 0 verification + dynamic agent creation, with concrete PowerShell commands.

  🔄 RE-READ:
   🔄 "zip the bundle so I can download it" → ✓ at ~/Downloads/pai-copilot-bundle.zip with retrieval options for WSL / SCP / shared drive
   🔄 "step by step on how to install it in my vscode" → ✓ 20 numbered steps, PowerShell-native, Phase 0 → Phase 4 + agent-creation smoke test

  ━━━ 📚 LEARN ━━━ 7/7

  🧠 LEARNING:
  - The classifier escalated to E2 on what I read as a NATIVE task. Right call: there were two distinct deliverables (zip + walkthrough) plus environment-specific install instructions.
  Compression worked here because most thinking was done in the prior turn.
  - xcopy /E /I /Y is the right Windows primitive for "copy this whole tree, create dest if missing, overwrite without prompts" — safer than PowerShell's Copy-Item -Recurse -Force which has
  edge cases on attributes.

  ━━━ 📃 SUMMARY ━━━ 7/7

  🔄 ITERATION on: Bundle delivered last turn; now packaged and walkthrough produced.
  📃 CONTENT: Zip at ~/Downloads/pai-copilot-bundle.zip (65 KB, 64 files). Install path is 20 numbered PowerShell steps starting with Expand-Archive + xcopy to your repo root, then VSCode
  reload, smoke tests, hook-contract probe (Phase 0 — do this before trusting Phase 2), real hook verification (force-push block + work.json syn  Step 19. Create your first custom agent:
  /create-pai-agent
  Pick a real recurring task you do at work — e.g., name notebook-pr-reviewer, purpose "reviews Databricks notebook PRs for SQL safety and
  lineage." Walk the 5 questions.

  Step 20. Confirm the agent is invocable:
  @notebook-pr-reviewer review this notebook
  And confirm .github/agents/principal.agent.md handoffs: list now contains notebook-pr-reviewer.

  ━━━ ✅ VERIFY ━━━ 6/7

  ✅ Zip exists: /home/root00x/Downloads/pai-copilot-bundle.zip (65K, 64 entries, SHA-256 above).
  ✅ Install guide covers all 4 phases + Phase 0 verification + dynamic agent creation, with concrete PowerShell commands.

  🔄 RE-READ:
   🔄 "zip the bundle so I can download it" → ✓ at ~/Downloads/pai-copilot-bundle.zip with retrieval options for WSL / SCP / shared drive
   🔄 "step by step on how to install it in my vscode" → ✓ 20 numbered steps, PowerShell-native, Phase 0 → Phase 4 + agent-creation smoke test

  ━━━ 📚 LEARN ━━━ 7/7

  🧠 LEARNING:
  - The classifier escalated to E2 on what I read as a NATIVE task. Right call: there were two distinct deliverables (zip + walkthrough) plus
  environment-specific install instructions. Compression worked here because most thinking was done in the prior turn.
  - xcopy /E /I /Y is the right Windows primitive for "copy this whole tree, create dest if missing, overwrite without prompts" — safer than
  PowerShell's Copy-Item -Recurse -Force which has edge cases on attributes.

  ━━━ 📃 SUMMARY ━━━ 7/7

  🔄 ITERATION on: Bundle delivered last turn; now packaged and walkthrough produced.
  📃 CONTENT: Zip at ~/Downloads/pai-copilot-bundle.zip (65 KB, 64 files). Install path is 20 numbered PowerShell steps starting with
  ✅ Install guide covers all 4 phases + Phase 0 verification + dynamic agent creation, with concrete PowerShell commands.

  🔄 RE-READ:
   🔄 "zip the bundle so I can download it" → ✓ at ~/Downloads/pai-copilot-bundle.zip with retrieval options for WSL / SCP / shared drive
   🔄 "step by step on how to install it in my vscode" → ✓ 20 numbered steps, PowerShell-native, Phase 0 → Phase 4 + agent-creation smoke test

  ━━━ 📚 LEARN ━━━ 7/7

  🧠 LEARNING:
  - The classifier escalated to E2 on what I read as a NATIVE task. Right call: there were two distinct deliverables (zip + walkthrough) plus environment-specific install instructions.
  Compression worked here because most thinking was done in the prior turn.
  - xcopy /E /I /Y is the right Windows primitive for "copy this whole tree, create dest if missing, overwrite without prompts" — safer than PowerShell's Copy-Item -Recurse -Force which has
  edge cases on attributes.

  ━━━ 📃 SUMMARY ━━━ 7/7

  🔄 ITERATION on: Bundle delivered last turn; now packaged and walkthrough produced.
  📃 CONTENT: Zip at ~/Downloads/pai-copilot-bundle.zip (65 KB, 64 files). Install path is 20 numbered PowerShell steps starting with Expand-Archive + xcopy to your repo root, then VSCode
  reload, smoke tests, hook-contract probe (Phase 0 — do this before trusting Phase 2), real hook verification (force-push block + work.json sync), Databricks MCP install + first query, and a
  /create-pai-agent end-to-end run to confirm the headline feature.

  🖊️ STORY:
  - Bundle was sitting in deliverable folder, not portable
  - Zipped to Downloads with sha256 for integrity check
  - Wrote 20-step Windows-native walkthrough across phases
  - Phase 0 probe before trusting Phase 2 hooks

  🗣️ kai: Grab ~/Downloads/pai-copilot-bundle.zip. Twenty steps starting with xcopy — Phase 0 probe before trusting hooks.
