# ComfyUI

## Session Continuity

At the start of every session:
1. Fetch `https://raw.githubusercontent.com/Mediajoy/Claude-snapshot/main/snapshots/ComfyUI/index.json` and read the `sessions` array
2. Show the user the last 3 sessions (timestamp + problemStatement)
3. Ask which to resume, or "start fresh" — wait for their answer
4. Load the chosen session by fetching `https://raw.githubusercontent.com/Mediajoy/Claude-snapshot/main/snapshots/ComfyUI/<file>` and reading `conversation`, `files`, and `problemStatement` as context

When the user says **"save snapshot"**, **"end session"**, **"sync session"**, or **"save progress"**:
1. Write the session file to `/Users/shahramsedehi/Documents/Github Local/claude-snapshot/snapshots/ComfyUI/<TIMESTAMP>.json` using this schema:
   ```json
   {
     "workspace": "ComfyUI",
     "conversation": [ last 20-30 relevant exchanges ],
     "files": [ { "path": "...", "state": "modified|created|deleted", "diff": "..." } ],
     "problemStatement": "Plain language summary of what was worked on and where things stand",
     "timestamp": "<ISO 8601>"
   }
   ```
2. Run: `bash "/Users/shahramsedehi/Documents/Github Local/claude-snapshot/scripts/save-snapshot.sh"`
3. Confirm the push succeeded

