---
layout: post
title: "VS Code: Run test at cursor automation"
author: Javier García
category: vscode
tags: vscode, elixir
---

TIL that with a simple task under `.vscode/tasks.json` I can automate running
the test under my editor cursor. Dead Simple.

```json
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "Elixir: Test at Cursor",
            "command": "mix test ${relativeFile}:${lineNumber}",
            "group": "test",
            "type": "shell",
            "problemMatcher": [
                "$mixCompileError",
                "$mixCompileWarning",
                "$mixTestFailure"
            ],
            "presentation": {
                "echo": true,
                "reveal": "always",
                "focus": false,
                "panel": "shared"
            }
        }
    ]
}
```

Now, if you're an automation-freak like me and you're not ok with just having
the task, but need a shortcut for it... then something like this just works:

```json
{
    "key": "ctrl+'",
    "command": "workbench.action.tasks.runTask",
    "args": "Elixir: Test at Cursor",
    "when": "editorLangId=='elixir'"
}
```

Happy coding :)
