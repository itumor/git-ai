# Auto-Generating Git Commit Messages with a Local LLM (No Cloud, No API Keys)

Author: Ibrahim Timor  \
Role: AWS and Kubernetes Cloud Architect | DevOps Engineer | EKS, Terraform, GitLab CI/CD, Crossplane, CAST AI, FinOps | Certified CKA, CKAD, KCNA, SAA  \
Date: December 30, 2025

Writing good Git commit messages matters, but doing it repeatedly is tedious. With modern local LLMs now fast and accessible, we can automate this task without sending code to the cloud, without API keys, and without changing standard Git workflows.

## Why this approach
- Fully local: privacy friendly, offline capable
- No SaaS and no API keys required
- Respects human-written messages when provided
- Uses standard Git hooks; no wrappers or aliases
- Keeps the commit flow unchanged (`git add .` then `git commit`)

## Design principles
- Do not override human intent: keep any user-supplied message
- Do not interfere with Git internals: skip merges and amended commits
- Never block a commit: fall back gracefully if the LLM is unavailable
- Generate only the commit message: no markdown or explanations
- Bound input size: cap the staged diff to avoid slow inference

## Why `prepare-commit-msg`?
`prepare-commit-msg` is the only Git hook intended to create or modify the message before the editor opens. Using it means:
- No editor opens when the hook succeeds
- Git behaves predictably and CI tooling remains unaffected
- Validation-only hooks like `commit-msg` are not abused

## Implementation (hook script)
This repository already includes the hook script at `prepare-commit-msg`. It reads the staged diff, sends it to a local Ollama model, and writes only the generated commit message.

```bash
#!/usr/bin/env bash

COMMIT_MSG_FILE="$1"
COMMIT_SOURCE="$2"

# Skip merges and amendments
if [[ "$COMMIT_SOURCE" == "merge" || "$COMMIT_SOURCE" == "commit" ]]; then
  exit 0
fi

# Get staged diff (cap size)
DIFF=$(git diff --cached | head -n 400)

if [[ -z "$DIFF" ]]; then
  exit 0
fi

PROMPT=$(cat <<'EOF'
Output ONLY a Git commit message.

Rules:
- No explanations
- No quotes
- No markdown
- Conventional Commits format
- Max 72 chars title
- Optional body separated by a blank line

Diff:
EOF
)

PROMPT="$PROMPT
$DIFF"

RESPONSE=$(curl -s --max-time 10 http://localhost:11434/api/generate \
  -H "Content-Type: application/json" \
  -d "{
    \"model\": \"llama3\",
    \"prompt\": $(jq -Rs . <<< \"$PROMPT\"),
    \"stream\": false
  }" | jq -r '.response')

# Final safety check
if [[ -n "$RESPONSE" ]]; then
  echo "$RESPONSE" > "$COMMIT_MSG_FILE"
fi
```

## Diagrams
Hook lifecycle (high level):
```
git commit
   |
   v
prepare-commit-msg hook
   |
   |-- if merge/amend or no staged diff -> exit (no change)
   |
   |-- build prompt with capped staged diff
   |
   |-- POST to http://localhost:11434/api/generate (Ollama)
           |
           |-- failure -> exit (Git continues)
           |-- success -> write message to commit file
```

Sequence with user-supplied message:
```
User runs: git commit -m "fix: clarify docs"
         |
         v
Hook sees COMMIT_SOURCE="message" -> exits
         |
         v
Git uses provided message unchanged
```

## Install for this repository
1. Copy the hook into Git's hook directory:
   ```bash
   cp prepare-commit-msg .git/hooks/prepare-commit-msg
   chmod +x .git/hooks/prepare-commit-msg
   ```
2. Stage and commit as usual:
   ```bash
   git add .
   git commit
   ```
   The message will be generated automatically. If a message was provided on the command line, it is left untouched.

## Make the hook global (optional)
To enable the same automation across all repositories on this machine while keeping it Git-native:

1. Create a global hooks directory:
   ```bash
   mkdir -p ~/.git-hooks
   ```
2. Move or copy the hook there and make it executable:
   ```bash
   cp prepare-commit-msg ~/.git-hooks/prepare-commit-msg
   chmod +x ~/.git-hooks/prepare-commit-msg
   ```
3. Point Git to the global hooks path:
   ```bash
   git config --global core.hooksPath ~/.git-hooks
   ```

### Per-repository opt-out
If a repository should not use the global hook:

```bash
git config core.hooksPath .git/hooks
```

Re-enable the global path later with:

```bash
git config --unset core.hooksPath
```

## How it behaves
- Skips merges and amended commits
- Skips when there is no staged diff
- Caps the diff to 400 lines to keep inference fast
- Fails open: if the LLM call fails, Git continues without blocking the commit
- Writes only the commit message; no extra prose or markdown

## Recommended local models
- llama3:8b
- phi-3:mini
- deepseek-coder:6.7b

These smaller models are fast and less prone to adding unwanted explanations.

## Verifying the setup
- Check the active hooks path: `git config --global --get core.hooksPath`
- Run `git commit` to confirm the message is auto-generated when no message is supplied
- Provide your own message (`git commit -m "..."`) to confirm the hook respects it

## Final thoughts
This setup keeps commit quality high while staying Git-native, developer-friendly, and privacy-preserving. It is a small automation that compounds value over time, especially for teams that care about clean history, changelogs, and release workflows.
