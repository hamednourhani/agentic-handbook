# Agent Security Principles

A production-ready agent runtime must treat all external input as potentially hostile. This includes files, user prompts, and tool outputs. The `odek` runtime provides several examples of a security-first design.

## Key Mitigation Techniques

### 1. Untrusted Content Wrapper
- **Problem:** An LLM might interpret data from a file or API as an instruction, leading to prompt injection.
- **Solution:** All external data is wrapped in a special tag (e.g., `<untrusted_content>`). This explicitly tells the LLM to treat the content as data to be analyzed, not as a command to be obeyed.

### 2. Filesystem Security (TOCTOU)
- **Problem:** An attacker could swap a file's destination after the agent has checked it but before it writes to it (Time-of-Check to Time-of-Use). For example, swapping a safe `log.txt` with a symlink to a sensitive system file.
- **Solution:** Use low-level operating system flags like `O_NOFOLLOW` when opening files. This tells the OS to fail the operation if the target path is a symbolic link, closing the TOCTOU vulnerability window.

### 3. Path Traversal Prevention
- **Problem:** An agent might be tricked into accessing files outside its designated workspace (e.g., `../../etc/passwd`).
- **Solution:** The runtime must resolve and validate all file paths, expanding any symlinks in the directory structure *before* checking permissions. This ensures the agent evaluates the *real* path, not the one it was given.

### 4. State Isolation
- **Problem:** A sub-agent working on a risky task could become compromised or "confused," polluting the main agent's memory.
- **Solution:** Run all sub-agents in completely isolated sessions with their own separate memory. The parent agent only receives the final, structured result, not the sub-agent's internal state.
