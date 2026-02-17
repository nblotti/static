---
name: session-filesystem
description: Workspace, context, and output folder conventions; retrieving externalized tool outputs.
allowed-tools: get_context
---
# Session filesystem contract

## Folders
- `workspace/` = working directory (default cwd for ALL commands)
- `context/` = persistent notes + externalized tool outputs
- `output/` = deliverables the user wants

## Path rules
- Use relative paths from `workspace/`.
- Do NOT use absolute `/shared/<thread_id>/...` paths.
- All file tools and `execute` are rooted in `workspace/`.
- Files written by any agent are visible to all others (shared NFS).

## Externalized outputs
If you see:
```
[Output externalized -> context/<filename> (N chars)]
```
Retrieve the full content via `get_context("<filename>")`.

## Best practices
- Write large intermediate results into `../context/` to keep future turns small.
- Put final artifacts in `../output/`.
- To send data to a subagent: write the file in `workspace/`, tell the subagent the relative path.
- To receive data from a subagent: tell it to save to a relative path, then `read_file` it.
