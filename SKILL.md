---
name: tmux
description: "Remote control tmux sessions for interactive CLIs (python, gdb, etc.) by sending keystrokes and scraping pane output."
license: Vibecoded
---

# tmux Skill

Use tmux as a programmable terminal multiplexer for interactive work. Works on Linux and macOS with stock tmux. Avoid using the user's default tmux socket to prevent interference; always use an isolated, private socket.

## <instructions>

You are equipped to use `tmux` to run and interact with long-running, stateful, or interactive command-line applications (like Python REPLs, debuggers, or persistent servers). 

When managing tmux sessions, strictly adhere to the following workflow:

### 1. Setup & Socket Convention
- **Always** use a dedicated socket to keep your sessions isolated from the user's personal tmux environment.
- Create and use the `AGENT_TMUX_SOCKET_DIR` (defaults to `${TMPDIR:-/tmp}/agent-tmux-sockets`).
- Default socket path: `SOCKET="$AGENT_TMUX_SOCKET_DIR/agent.sock"`.

```bash
# Standard setup snippet
SOCKET_DIR=${TMPDIR:-/tmp}/agent-tmux-sockets
mkdir -p "$SOCKET_DIR"
SOCKET="$SOCKET_DIR/agent.sock"
SESSION=agent-python # Use descriptive, slug-like names without spaces
```

### 2. Session Management
- **Target format**: `{session}:{window}.{pane}`, defaults to `:0.0` if omitted. Keep names short (e.g., `agent-py`, `agent-gdb`).
- **Start**: `tmux -S "$SOCKET" new -d -s "$SESSION" -n shell`
- **Inspect**: 
  - `tmux -S "$SOCKET" list-sessions`
  - `tmux -S "$SOCKET" list-panes -a`
- **Clean up**: `tmux -S "$SOCKET" kill-session -t "$SESSION"`

### 3. User Visibility (CRITICAL)
After starting a session, **ALWAYS** tell the user how to monitor the session by giving them a command to copy-paste. This must be printed immediately after the session is created.

```
To monitor this session yourself:
  tmux -S "$SOCKET" attach -t <session-name>

Or to capture the output once:
  tmux -S "$SOCKET" capture-pane -p -J -t <session-name>:0.0 -S -200
```

### 4. Sending Input Safely
- Prefer literal sends to avoid shell splitting: `tmux -S "$SOCKET" send-keys -t "$SESSION" -l -- "$cmd"`
- When composing inline commands, use single quotes or ANSI C quoting to avoid expansion: `tmux -S "$SOCKET" send-keys -t "$SESSION" -- $'python3 -m http.server 8000'`
- To send control keys: `tmux -S "$SOCKET" send-keys -t target C-c`, `C-d`, `C-z`, `Escape`, `Enter`.

### 5. Watching Output & Synchronization
- **Capture recent history**: `tmux -S "$SOCKET" capture-pane -p -J -t "$SESSION" -S -200`
- **Poll for prompts**: Use timed polling to avoid races with interactive tools. 
  Example: Wait for a Python prompt before sending code:
  ```bash
  ./scripts/wait-for-text.sh -S "$SOCKET" -t "$SESSION":0.0 -p '^>>>' -T 15 -l 4000
  ```

### 6. Interactive Tool Recipes
- **Python REPL**: 
  - Send: `tmux -S "$SOCKET" send-keys -t "$SESSION" -- 'PYTHON_BASIC_REPL=1 python3 -q' Enter`
  - Wait: `wait-for-text.sh` for `^>>>`
  - *Note: `PYTHON_BASIC_REPL=1` is critical to prevent the console from interfering with `send-keys`.*
- **GDB/LLDB**: 
  - Send: `tmux -S "$SOCKET" send-keys -t "$SESSION" -- 'gdb --quiet ./a.out' Enter`
  - Disable paging: `tmux -S "$SOCKET" send-keys -t "$SESSION" -- 'set pagination off' Enter`
  - Break: send `C-c`
  - Exit: send `quit` then `y`.
  - *Note: Default to `lldb` if debugging on macOS.*

### 7. Helper Scripts
You have access to helper scripts in the `./scripts` directory:

- **`find-sessions.sh`**: List sessions with metadata.
  - Scan active socket: `./scripts/find-sessions.sh -S "$SOCKET"`
  - Filter: `./scripts/find-sessions.sh -S "$SOCKET" -q "python"`
- **`wait-for-text.sh`**: Poll a pane for a regex (or fixed string) with a timeout.
  ```bash
  ./scripts/wait-for-text.sh -S "$SOCKET" -t session:0.0 -p 'pattern' [-F] [-T 20] [-i 0.5] [-l 2000]
  ```

### 8. Teardown
- Kill specific session: `tmux -S "$SOCKET" kill-session -t "$SESSION"`
- Wipe completely: `tmux -S "$SOCKET" kill-server`
</instructions>
