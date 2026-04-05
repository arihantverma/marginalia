---
title: "Killing a Process That Is Holding a Local Development Port"
date: 2026-04-05
tags: ["ports", "networking", "lsof", "kill", "macos", "zsh", "pnpm", "debugging", "local-development"]
---

## Context

I ran `pnpm db:dev` for a local libSQL/Turso development server and got an error that the port was already in use. In this project, `pnpm db:dev` starts `turso dev` on port `8080`, so the immediate problem was to find which process was already listening on `8080`, terminate it, and verify the port was actually free before rerunning the command.

## Learnings

### How to Find the Process Using a Port

On macOS, `lsof` is the most direct tool for this:

```bash
lsof -nP -iTCP:8080 -sTCP:LISTEN
```

What each part means:

- `lsof`: list open files; on Unix-like systems, network sockets are treated like files
- `-nP`: show numeric addresses and ports instead of doing DNS/service-name lookups
- `-iTCP:8080`: filter to TCP sockets on port `8080`
- `-sTCP:LISTEN`: only show processes that are actively listening on that port

Example output:

```text
COMMAND   PID         USER   FD   TYPE   DEVICE SIZE/OFF NODE NAME
sqld    23283 arihantverma    9u  IPv4  ...         0t0  TCP *:8080 (LISTEN)
```

The key field is `PID`. That is the process ID you can signal with `kill`.

If I only want the PID and not the full table:

```bash
lsof -tiTCP:8080 -sTCP:LISTEN
```

That prints just the process ID, which makes it useful in scripts or command substitution.

### How to Stop the Process

The usual sequence is:

```bash
kill <PID>
```

Example:

```bash
kill 23283
```

This sends `SIGTERM`, which is the polite request for the process to shut down. It gives the process a chance to clean up.

If the process does not exit and is still holding the port, escalate to:

```bash
kill -9 <PID>
```

Example:

```bash
kill -9 23283
```

`-9` sends `SIGKILL`, which the process cannot ignore. This is more forceful, so it should generally be the second step, not the first.

### Why Verification Matters

After sending a signal, do not assume the process is gone. Check again:

```bash
lsof -nP -iTCP:8080 -sTCP:LISTEN
```

If the command returns no rows, the port is no longer being listened on. That is the signal that I can safely rerun the original dev command.

This matters because:

- `kill` may fail due to permissions
- a process may ignore or delay handling `SIGTERM`
- a supervisor may automatically restart the process
- I might have targeted the wrong PID

### One-Liner Versions

If I want a fast one-liner:

```bash
kill $(lsof -tiTCP:8080 -sTCP:LISTEN)
```

If that is not enough:

```bash
kill -9 $(lsof -tiTCP:8080 -sTCP:LISTEN)
```

These are convenient, but they are slightly less educational than doing the lookup first, reading the output, and then deciding how aggressively to terminate the process.

### A Good Habit for Local Development

Use this sequence:

1. Check who owns the port.
2. Send a normal `kill`.
3. Recheck the port.
4. Only then use `kill -9` if the process is still listening.

That gives a cleaner and safer workflow than immediately force-killing everything.

### How This Applied in Practice

For this specific case:

- `pnpm db:dev` in the portfolio project uses port `8080`
- `lsof` showed `sqld` was already listening there
- a normal `kill` did not free the port
- `kill -9` did free the port
- a final `lsof` check returned nothing, which confirmed the port was available again

## References

- `man lsof`
- `man kill`
- [lsof FAQ](https://lsof.readthedocs.io/en/stable/faq/)
