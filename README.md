# IBM i Agent Bridge

A VS Code extension for GitHub Copilot Chat and Code for IBM i. It reuses the active IBM i connection and contributes tools for reading and proposing changes to source members. It has no runtime npm dependencies, stores no IBM i credentials, and does not issue compile commands. Source returned by a tool is available to the selected AI model. Use your organization's approved model and source scope.

## Install and configure

1. Use desktop VS Code 1.105 or newer with Code for IBM i and GitHub Copilot Chat installed. Sign in to Copilot and connect to a DEV system in Code for IBM i.
2. Run **Extensions: Install from VSIX…** from the Command Palette and select the package.
3. Open **Preferences: Open User Settings (JSON)**. Add these keys alongside your other settings and replace the examples with the exact DEV connection name in Code for IBM i and your DEV library name. An empty connection name or list disables patch proposals and apply.

```json
{
  "ibmiAgentBridge.allowedWriteConnection": "MY-IBM-I-DEV",
  "ibmiAgentBridge.allowedWriteLibraries": ["DEVLIB"],
  "files.autoSave": "off"
}
```

4. Open an editable test member from the Code for IBM i **Object Browser**, then open Copilot Chat. Select a local Agent session that exposes VS Code extension tools. Enable the three IBM i Bridge tools in the Chat tool picker if needed.

## Read and edit

- **`#ibmiReadActive`** reads the active member editor, including unsaved text.
- **`#ibmiReadMember`** reads a member by library, source physical file and member name without its extension. Read access follows the active IBM i profile and is independent of the write allowlist.
- **`#ibmiProposePatch`** accepts a snapshot ID from the active-member tool and exact, unique text replacements. A proposal requires a clean, writable editor and opens a read-only diff.

Try this on a disposable DEV member:

> Use #ibmiReadActive to read the open member. Use #ibmiProposePatch to change the text of one existing comment to AI edit test. Keep the original comment marker and fixed-format columns.

Review the diff, then run **IBM i Agent Bridge: Apply Reviewed Patch** or **IBM i Agent Bridge: Discard Patch** from the Command Palette. Apply updates the editor buffer after confirmation. Review the editor and press **Ctrl+S** yourself to save through Code for IBM i. Reopen the member to verify it on the server. The bridge checks the connection, editor version, and the provider's saved source at preview and apply. It does not lock the member, so coordinate concurrent changes before saving.

Patch proposals in this version target standard `member:` documents with `LIBRARY/SOURCEFILE/MEMBER` paths. IFS streamfiles and ASP-prefixed member paths are outside its patch scope. If the preview reports that the editor and remote source differ, reopen the member and retry. Check RPG fixed-format columns, source record length and compile in DEV using your existing workflow.

## Build

The extension is plain JavaScript. With Node.js installed, run `node --check extension.js` and then `npx @vscode/vsce@4.1.0 package --no-dependencies` from the project directory.
