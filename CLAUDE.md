# Claude Code — client-sfl

## Pushing to GitHub

The local git proxy is read-only (`CCR_TEST_GITPROXY=1` returns 403 on push),
and the GitHub MCP token lacks write access. Use the DevOps VM's SSH deploy key instead.

### Quick Reference (3 steps)

```bash
# 1. Generate patch from local commits
git format-patch --stdout origin/main > /tmp/changes.patch

# 2. Transfer patch to VM via hex encoding (survives ssh_exec without corruption)
python3 -c "print(open('/tmp/changes.patch','rb').read().hex())" > /tmp/p.hex
# Then write hex to VM in chunks of ~10000 chars using ssh_exec:
#   ssh_exec: python3 -c "open('/tmp/p.hex','w').write('<chunk1>')"
#   ssh_exec: python3 -c "open('/tmp/p.hex','a').write('<chunk2>')"
#   ... repeat for all chunks ...

# 3. On VM: decode, apply, push
ssh_exec: python3 -c "import binascii; open('/tmp/changes.patch','wb').write(binascii.unhexlify(open('/tmp/p.hex').read()))"
ssh_exec: sudo -u claude-bridge git -C /home/claude-bridge/client-sfl am /tmp/changes.patch
ssh_exec: sudo -u claude-bridge git -C /home/claude-bridge/client-sfl push -u git@github-client-sfl:necklesio/client-sfl.git <branch-name>
```

### Details

- **VM**: `dev-ops-neckles-io-u50517` via `ssh_exec` (clientCode: `NIO`, accessLevel: `VM`)
- **Push user**: `claude-bridge` (has SSH deploy key for this repo)
- **SSH host alias**: `github-client-sfl` → github.com with `id_ed25519_client_sfl` identity
- **Clone location on VM**: `/home/claude-bridge/client-sfl/`
- **Encoding**: Use hex (`bytes.hex()`) — only `[0-9a-f]` chars, immune to corruption. Avoid base64 (introduces `-` and `\x00` via ssh_exec).
- **Chunk size**: ~10000 hex chars per ssh_exec call to avoid timeouts.
- **VM repo setup** (if needed):
  ```bash
  sudo -u claude-bridge git -C /home/claude-bridge/client-sfl fetch origin
  sudo -u claude-bridge git -C /home/claude-bridge/client-sfl checkout -b <branch-name>
  ```

## Project Stack

- Static HTML site: Tailwind CSS + Alpine.js v3 (CDN)
- No build step — edit `index.html` directly
- Alpine.js patterns: `x-data`, `x-model`, `x-show`, `x-transition`, `$dispatch`, `@event.window`
