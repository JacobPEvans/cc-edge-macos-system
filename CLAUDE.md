# cc-edge-macos-system

Cribl Edge pack for macOS system health monitoring and anomaly detection.

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
   tar -czf cc-edge-macos-system-vX.Y.Z.crbl data default package.json README.md
   cp cc-edge-macos-system-vX.Y.Z.crbl cc-edge-macos-system.crbl
   gh release create vX.Y.Z --draft \
     --repo JacobPEvans/cc-edge-macos-system \
     cc-edge-macos-system-vX.Y.Z.crbl cc-edge-macos-system.crbl
   gh release edit vX.Y.Z --repo JacobPEvans/cc-edge-macos-system --draft=false
   ```

## File Operations

- Read files with the Read tool (NEVER cat, head, tail)
- Edit existing files with the Edit tool (NEVER sed, awk)
- Create new files with the Write tool (NEVER echo >, heredocs)
- Search file contents with the Grep tool
- Find files with the Glob tool
