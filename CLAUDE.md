# cc-edge-the-mac-pack-io

Cribl Edge pack for comprehensive macOS system, power, and performance monitoring.

The pack targets a **native macOS Cribl Edge install** (Nix-managed). Exec
inputs that invoke macOS-only binaries (`powermetrics`, `pmset`, `ioreg`,
`memory_pressure`) require host access and will not work from a Linux
container. The file input (`mac-perf-snapshots`) reads from a Mac
filesystem path and likewise requires the native install. An OrbStack-
deployed Cribl Edge — used elsewhere for cluster monitoring — is unrelated
to and not used by this pack.

## Version Policy

This pack uses **semantic versioning**.

### Rules for AI agents

- **Patch bumps** (`X.Y.Z → X.Y.Z+1`): AI may bump for bug fixes, performance improvements, and minor corrections.
- **Minor bumps** (`X.Y.z → X.Y+1.0`): AI may bump for new features, new inputs, or non-breaking changes.
- **Major bumps** (`X.y.z → X+1.0.0`): **Human only.** Never bump the major version without explicit human approval.

### Release workflow

1. Make changes on a feature branch
2. Update `package.json` version (minor or patch only)
3. Update the `## Release Notes` section in `README.md` with the new version entry
4. Merge PR to main
5. Create GitHub release with the `.crbl` artifact built locally:

   ```sh
   tar -czf cc-edge-the-mac-pack-io-vX.Y.Z.crbl data default package.json README.md
   cp cc-edge-the-mac-pack-io-vX.Y.Z.crbl cc-edge-the-mac-pack-io.crbl
   gh release create vX.Y.Z --draft \
     --repo JacobPEvans/cc-edge-the-mac-pack-io \
     cc-edge-the-mac-pack-io-vX.Y.Z.crbl cc-edge-the-mac-pack-io.crbl
   gh release edit vX.Y.Z --repo JacobPEvans/cc-edge-the-mac-pack-io --draft=false
   ```

## File Operations

- Read files with the Read tool (NEVER cat, head, tail)
- Edit existing files with the Edit tool (NEVER sed, awk)
- Create new files with the Write tool (NEVER echo >, heredocs)
- Search file contents with the Grep tool
- Find files with the Glob tool
