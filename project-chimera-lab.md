# Lab Walkthrough: Project Chimera (Linux)

This guide provides a step-by-step solution to the Project Chimera lab using `gdb` and `/proc`.

> [!WARNING]
> SPOILER ALERT: Do not read this if you prefer to solve the lab on your own!

## Phase 1: Bypassing PTRACE Anti-Debugging

The `system-update` binary employs `ptrace(PTRACE_TRACEME)` to detect if it's being debugged. If `gdb` or `strace` is attached, this system call returns `-1`.

1. Open it in GDB: `gdb ./bin/system-update`.
2. Run `catch syscall ptrace` to set a catchpoint on the `ptrace` system call.
3. Type `run`. When the program stops at the catchpoint (entering the syscall), type `continue` to step *out* of the kernel and back into user space (leaving the syscall).
4. Once GDB pauses leaving the syscall, inspect the return value register: `info registers rax`. It will be `-1` (0xffffffffffffffff).
5. **The Patch:** Change the return value from `-1` (error) to `0` (success) by typing: `set $rax = 0`.
6. `continue` execution. The binary now believes it is not being debugged and will proceed to the dropper phase!

## Phase 2: Identifying Fileless Execution

Once the anti-debugger is bypassed:
1. Continue observing execution. You will observe a call to `memfd_create("payload_mem", ...)`. This creates a file in RAM that has no corresponding path on disk.
2. The binary then calls `write` multiple times, pushing a bash script into this anonymous file descriptor (FD).
3. The lab explicitly pauses before executing the payload to allow for extraction. In a real-world scenario, it would immediately call `fexecve`.

## Phase 3: Extraction

To grab the payload before it runs:
1. When the binary pauses and asks you to dump the file, background your GDB process using `CTRL+Z` (or open a new terminal tab).
2. Find the Process ID (PID) of the suspended binary using `ps aux | grep system-update`. (Let's assume PID `1042`).
3. Navigate to `/proc/1042/fd/`. You will see numerical links to different files open by the process.
4. `ls -l` the directory. One of the file descriptors (e.g., `3` or `4`) will link to `/memfd:payload_mem (deleted)`.
5. **Dump it:** Copy it securely to your home folder: `cat /proc/1042/fd/3 > ~/payload_dumped.sh`.
6. You have successfully recovered the fileless payload! You can open `payload_dumped.sh` to view the mock C2 banner script.
