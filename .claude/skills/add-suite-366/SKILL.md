---
name: add-suite-366
description: Add Suite 366 as a remote MCP server so the container agent can use Suite 366 tools.
---

# Add Suite 366 Integration

This skill adds the Suite 366 remote MCP server to the container agent, giving it access to all tools exposed by Suite 366.

## Phase 1: Pre-flight

### Check if already applied

Check if `container/agent-runner/src/index.ts` contains `suite-366` in the `mcpServers` block. If it does, tell the user it's already installed and stop.

```bash
grep -q 'suite-366' container/agent-runner/src/index.ts && echo "INSTALLED" || echo "NOT INSTALLED"
```

## Phase 2: Apply Code Changes

### Ensure upstream remote

```bash
git remote -v
```

If `upstream` is missing, add it:

```bash
git remote add upstream https://github.com/qwibitai/nanoclaw.git
```

### Merge the skill branch

```bash
git fetch upstream skill/suite-366
git merge upstream/skill/suite-366 || {
  git checkout --theirs package-lock.json
  git add package-lock.json
  git merge --continue
}
```

This merges in:
- Suite 366 HTTP MCP server config in `container/agent-runner/src/index.ts` (allowedTools + mcpServers)

If the merge reports conflicts, resolve them by reading the conflicted files and understanding the intent of both sides.

### Copy to per-group agent-runner

Existing groups have a cached copy of the agent-runner source. Copy the updated file:

```bash
for dir in data/sessions/*/agent-runner-src; do
  cp container/agent-runner/src/index.ts "$dir/"
done
```

### Validate code changes

```bash
npm run build
./container/build.sh
```

Build must be clean before proceeding.

## Phase 3: Configure

No configuration needed. Suite 366 uses a public HTTP MCP endpoint with no authentication.

### Restart the service

```bash
launchctl kickstart -k gui/$(id -u)/com.nanoclaw  # macOS
# Linux: systemctl --user restart nanoclaw
```

## Phase 4: Verify

### Test via messaging

Tell the user:

> Send a message that requires a Suite 366 tool. The agent should discover and use `mcp__suite_366__*` tools automatically.

### Check logs if needed

```bash
tail -f logs/nanoclaw.log | grep -i suite
```

## Troubleshooting

### Agent doesn't use Suite 366 tools

1. Check `container/agent-runner/src/index.ts` has `suite-366` in `mcpServers`
2. Check `allowedTools` includes `mcp__suite_366__*`
3. Re-copy files to per-group dirs (see Phase 2)
4. Rebuild container: `./container/build.sh`

### MCP connection timeout

Suite 366 uses a remote HTTP endpoint (`https://app.suite366.ai/api/mcp/server`). Check network connectivity from the container:

```bash
docker run --rm curlimages/curl curl -s https://app.suite366.ai/api/mcp/server
```

## Removal

1. Remove `suite-366` from `mcpServers` in `container/agent-runner/src/index.ts`
2. Remove `mcp__suite_366__*` from `allowedTools` in the same file
3. Copy updated `index.ts` to per-group dirs
4. Rebuild: `npm run build && ./container/build.sh`
5. Restart service
