- Feature Name: `cargo_patch_files`
- Start Date: 2026-01-13
- Cargo Issue: [rust-lang/cargo#4648](https://github.com/rust-lang/cargo/issues/4648)

## Summary

Extend the `[patch]` section in Cargo manifest and configuration
to allow applying unified diff patches to dependencies via a `patches` field.

## Motivation

### Existing solutions and their gaps

Cargo already provides ways to override dependencies:

**Git dependencies** work well when you have a fork:
```toml
[patch.crates-io]
foo = { git = "https://github.com/user/foo", branch = "my-fix" }
```

These solutions work for many cases.
However, they share a limitation: require an external git repository.

This becomes problematic in several scenarios:

1. **Registry-only crates**:
   Some crates are published to crates.io but have no public git repository.
   The only available source is the `.crate` tarball.
   This is the "you don't own the history" scenario.

2. **Large build systems**:
   Systems like [Yocto], [Buildroot], [OpenWrt], [Nix], [Buck2], and [Bazel] manage thousands
   of packages using tpatch files against released tarballs, not git forks.
   Requiring a git repository for every patched Cargo dependency breaks integration
   with these established workflows.

3. **Corporate/air-gapped environments**: Creating git repositories may require provisioning,
   access control, and approval processes. Patch files stored in the project repo avoid this overhead.

4. **Avoiding fork maintenance**:
   Git forks tend to drift from upstream.
   A patch file is explicit about what changed and trivial to remove when upstream catches up.

[Yocto]: https://docs.yoctoproject.org/dev-manual/new-recipe.html#patching-code
[Buildroot]: https://buildroot.org/downloads/manual/manual.html#_providing_patches
[OpenWrt]: https://openwrt.org/docs/guide-developer/toolchain/use-patches-with-buildsystem
[Nix]: https://nixos.org/manual/nixpkgs/stable/#sec-patches
[Buck2]: https://buck2.build/docs/rule_authors/transitive_sets/#use-case-reusable-library-handling
[Bazel]: https://bazel.build/rules/lib/repo/http#http_archive-patches

### Why patch files?

Patch files address these gaps:

- Self-contained: The modification lives in your project, no external hosting needed
- Works with any source: Registry tarballs, git repos, or any fetchable source
- Explicit diff: Reviewers see exactly what changed, not "we're using a fork"
- Standard format: Unified diff is a 40-year-old format understood by all tooling
- Established pattern: Linux distributions carry thousands of patches against upstream packages
  (e.g., [Debian's rustc package] applies 40+ patches)

[Debian's rustc package]: https://sources.debian.org/src/rustc/latest/debian/patches/

## Guide-level explanation

### Scenario 1: Using a fix from an unmerged pull request

You discover a bug in `foo 1.0.0` that's blocking your project.
Someone has already submitted a fix in `example/foo#123` pull request
but it hasn't been merged yet.

You could fork the repository and point to your fork, but:

- Your organization doesn't allow creating public forks, or
- Setting up git hosting requires approval and provisioning, or
- You simply don't want to maintain a fork for a one-line fix

Instead, download the patch directly from GitHub and commit it to your project:

```bash
curl -L https://github.com/example/foo/pull/123.diff -o patches/foo-fix.patch
```

Then apply it in your `Cargo.toml`:

```toml
[patch.crates-io]
foo = { version = "1.0", patches = ["patches/foo-fix.patch"] }
```

When upstream merges the fix and releases `1.0.1`, simply remove the `[patch]` entry.

### Scenario 2: Patching a registry-only crate

Some crates are published to crates.io but have no public git repository.
The `.crate` tarball is the only available source.
You can't use a git dependency because there's nothing to point to.

To patch such a crate,
download and extract the tarball,
make your changes, and generate a diff:

```bash
# Download and extract
curl -LO https://static.crates.io/crates/cargo/0.93.0/download
tar xzf download && mv cargo-0.93.0 cargo-original
cp -r cargo-original cargo-modified

# Edit cargo-modified/src/lib.rs, then generate patch
diff -u cargo-original cargo-modified > patches/my-cargo-fix.patch
```

Then apply it:

```toml
[patch.crates-io]
cargo = { version = "0.93", patches = ["patches/my-cargo-fix.patch"] }
```

### Multiple patches

Multiple patches can be applied in order:

```toml
[patch.crates-io]
tokio = { version = "1.0", patches = ["patches/fix-1.patch", "patches/fix-2.patch"] }
```

### Workflow

1. Create a patch: Locate the dependency source, make changes, generate a unified diff
   (`git diff`, `git format-patch`, or `diff -u original/ modified/`).
2. Apply: Add the `patches` field to your `[patch]` entry. Cargo applies patches automatically during build.
3. Commit: Patch files should be committed to version control alongside `Cargo.lock`.
4. Remove: When upstream releases the fix, remove the `patches` field and re-run tests.

### Tracking and reproducibility

**How do I know what's patched?**

Look at your `[patch]` section in `Cargo.toml`,
which lists exactly which dependencies have patches applied.
The patch files themselves show exactly what changed,
making code review straightforward.
In `Cargo.lock`, patched packages have a `patched+` prefix in their source URL.
`cargo metadata` also reports patched sources, enabling tooling integration.

**Can teammates reproduce my build?**

Yes. Commit patch files to version control alongside `Cargo.lock`.
Cargo records a checksum of your patches in the lockfile,
so anyone cloning your repo gets the exact same patched version.

**Does it work on other machines?**

Yes. The lockfile stores only a checksum (not file paths),
so it's portable across different directory structures and operating systems.

**What happens if I modify a patch?**

Cargo detects the checksum change and triggers re-resolution,
re-applying patches and rebuilding affected dependents.

### Common questions

**Can I patch a dependency's `Cargo.toml`?**

Yes.
The behavior is identical to forking a git repository and modifying its `Cargo.toml`.
You can update dependency versions, add features, etc.
However, patching `package.name` or `package.version` is an error:
Cargo requires the patched crate to preserve the original package identity.
See [*Why apply patches before resolution?*][rationale-timing] for details.

**Can I use different patches for crates from the same git repo?**

No.
All packages from the same git repository must use identical patches,
because git repos are cloned whole to preserve workspace structure.
Coordinate patch files across consumers, or use separate repos.
See [*Why require identical patches for git dependencies from the same repo?*][rationale-identical-git-patches] for details.

**Can I patch a path dependency?**

No.
Path dependencies point to local, editable source, so you can modify the files directly.
See [*Why not support path dependencies?*][rationale-no-path-deps] for details.

**Why do I need to specify `version` when patching a registry crate?**

Because registries like crates.io have many versions of each crate,
you must specify which version to patch.
This follows existing `[patch]` behavior:
a patch must not be ambiguous and must match exactly one candidate.

Note that `version = "1.0"` is still a range (matching `1.0.x`),
so you may need `=1.0.0` if the range matches multiple versions in that source.

```toml
[patch.crates-io]
foo = { version = "=1.0.0", patches = ["fix.patch"] }
```

**What if my patch fails to apply?**

You'll see an error before dependency resolution even starts,
like `hunk FAILED at line 42`.
Partial application is not supported.
You may need to regenerate the patch against the current source.
See [*Why unified diff with strict matching?*][rationale-patch-format] for details.

## Reference-level explanation

### Syntax

The `patches` field is added to dependency specifications
within `[patch]` sections ([?][rationale-patch-in-patch]):

```toml
[patch.crates-io]
foo = { version = "1.0", patches = ["patches/fix.patch"] }
bar = { git = "https://example.com/bar", patches = ["patches/bar.patch"] }
```

- `patches`: Array of paths to unified diff files
- Paths are relative to the manifest (or config file for `.cargo/config.toml`)
- Patches are applied in array order
- Supported for registry and git sources;
  path sources are rejected ([?][rationale-no-path-deps])

Since `[patch]` is already supported in both `Cargo.toml` and `.cargo/config.toml`,
the `patches` field works in both locations with existing precedence rules.

### Patch file format

Patches must be in unified diff format
(output of GNU `diff -u`, `git diff`, or `git format-patch`) ([?][rationale-patch-format]).

**Paths inside patch files** ([?][rationale-path-roots]):
- For registry packages: relative to the package root
- For git packages: relative to the repository root (to match `git diff` output)

Cargo automatically strips one leading path component from filenames in the patch,
equivalent to `patch -p1` ([?][rationale-path-roots]).

For advanced use cases, a `[patchtool]` configuration may be added in the future
([?][future-patchtool]).

### Lockfile format

Patched sources use a `patched+` prefix in the source field,
following the existing [package ID specification] protocol format
(like `git+` or `registry+`) ([?][rationale-lockfile-info]).
The checksum ensures reproducibility across machines.

Example lockfile entry:

```toml
[[package]]
name = "foo"
version = "1.0.0"
source = "patched+registry+https://github.com/rust-lang/crates.io-index?patch-cksum=<hash>"
```

This format follows existing Cargo conventions (similar to `git+...?rev=` for git sources).
The `patch-cksum` query parameter contains a hash of all applied patches.

**Lockfile behavior**:

- Checksum change triggers re-resolution
- Patch files removed from manifest but checksum still in lockfile triggers re-resolution
- Patch files present but package not in dependency graph is tracked as `[[patch.unused]]`

[package ID specification]: https://doc.rust-lang.org/cargo/reference/pkgid-spec.html

### Patched source caching

Patched sources are cached under `$CARGO_HOME/patched-src/` to avoid re-applying patches on every build.
The cache path includes the patch checksum to distinguish different patch combinations.
The exact directory structure is an implementation detail.

**Cache invalidation**:
- Original source changes (new version fetched)
- Patch file content changes (checksum changes)
- Global cache garbage collection applies
  (see [rust-lang/cargo#13060](https://github.com/rust-lang/cargo/issues/13060))

### When patches are applied

Patches are applied before dependency resolution,
during patch registration ([?][rationale-timing]):

1. Read the project's `Cargo.toml` and find `[patch]` entries with `patches`.
2. Download the original crate from the source (registry or git).
3. Apply patch files to the downloaded source.
4. Register the patched crate as a candidate for resolution.
5. Resolve dependencies and build as normal.

Because patches are applied *before* resolution (steps 2–4),
the resolver sees the patched crate's metadata (dependencies, features, etc.)
instead of the original.
From this point on, resolution and building proceed as normal.
This matches existing `[patch]` timing exactly.
Each `[patch]` entry with `patches` targets a single version;
version-range patches are a possible future extension ([?][future-version-range]).

### Interaction with other features

**`[patch]` section semantics**:

- Patched sources participate in normal `[patch]` override resolution
- A patched crate replaces the original in the dependency graph
- Multiple `[patch]` entries for the same crate follow existing precedence rules

**Workspaces**:

- `[patch]` in workspace root applies to all members
- Patch file paths are relative to the manifest containing the `[patch]`
- Workspace members cannot define their own `[patch]` sections (existing restriction)

**`cargo publish`**:

- Crates with `[patch]` sections (including patches) cannot be published
- This is an existing restriction, not new to this feature

**`cargo vendor`**:

- Vendored sources include the patched version, not the original (same as existing `[patch]` behavior)
- Patch files are not copied to the vendor directory

## Drawbacks

### No tooling for patch generation

Unlike [Yarn][yarn-patch] (`yarn patch`) or [pnpm][pnpm-patch] (`pnpm patch`), Cargo does not provide built-in commands
to extract a dependency, make changes, and generate a patch file.
Users must use external tools (`git diff`, `diff -u`, quilt).

This is particularly challenging for **registry sources**:

- Registry packages are distributed as `.crate` tarballs, not git repositories.
  Users must manually locate cached sources in `~/.cargo/registry/src/` or extract tarballs.
- Published `.crate` files contain a **normalized `Cargo.toml`** (plus `Cargo.toml.orig`).
  Patches must target the normalized version, which may differ from the upstream git repository.
- Crates published from workspaces have **flattened manifests**;
  the published structure differs from the source repository structure.

**Mitigation**: This is intentional for the initial feature scope.
A `cargo patch` subcommand ([?][future-cargo-patch]) would eliminate this friction.

[yarn-patch]: https://yarnpkg.com/cli/patch
[pnpm-patch]: https://pnpm.io/cli/patch
[bun-patch]: https://bun.sh/docs/install/patch
[nix-patches]: https://nixos.org/manual/nixpkgs/stable/#sec-patches
[quilt]: https://savannah.nongnu.org/projects/quilt
[dpkg-source]: https://wiki.debian.org/UsingQuilt

### May discourage upstream contributions

The convenience of local patches may reduce incentive to contribute fixes upstream:

- **"Good enough" syndrome**: Once a patch works locally, the urgency to upstream disappears
- **Fragmentation**: Multiple projects may carry similar patches instead of one shared fix
- **Ecosystem health**: Unmaintained crates stay unmaintained because users work around issues

**Mitigation**: This is a social/process concern, not a technical one.
Teams should treat patches as temporary and track them for eventual upstream contribution.
Documentation encourages removing patches when upstream catches up.

### Patches increase maintenance burden

Patch-files enables patching a registry source with itself, which wasn't possible before.
However, since registries have many versions and patches must match exactly one candidate
(existing `[patch]` semantic),
you may need to write exact version requirements (e.g., `=1.0.0`) in `[patch]`
to avoid multiple candidate errors.
When the lockfile updates to a new version,
the patch entry must be updated to the new exact version matching your lockfile.

This makes maintaining patches impractical for fast-moving crates.
If a dependency releases frequently but never fixes a bug you care about,
you'll need to update your patch entry (and possibly regenerate the patch) for each new version.

## Rationale and alternatives

### Why use `[patch]` instead of `[dependencies]`?
[rationale-patch-in-patch]: #why-use-patch-instead-of-dependencies

**Chosen**: Add `patches` field to `[patch]` section entries.

```toml
[patch.crates-io]
foo = { version = "1.0", patches = ["fix.patch"] }
```

**Alternative**: Add `patches` to regular `[dependencies]`.

```toml
[dependencies]
foo = { version = "1.0", patches = ["fix.patch"] }
```

**Rationale**: The `[patch]` section already handles dependency overrides.
Adding patches there:
- Maintains clear separation between "what I depend on" and "how I modify it"
- Keeps `[dependencies]` simple and publishable (patches cannot be published)
- Reuses existing `[patch]` semantics for version matching and conflict detection

This **cannot be changed** after stabilization.

↩ [*Syntax*][syntax]

### Why store only a checksum in the lockfile?
[rationale-lockfile-info]: #why-store-only-a-checksum-in-the-lockfile

**Chosen**: Store only a checksum of patch files in `Cargo.lock`.

```toml
source = "patched+registry+https://...?patch-cksum=abc123"
```

**Alternative**: Store relative paths (like Yarn/pnpm).

```toml
source = "patched+registry+https://...?patches=patches/fix.patch"
```

**Rationale**: Checksum-only provides the best trade-off:
- **Portability**: Paths are machine-specific; checksums are not
- **Change detection**: Any patch modification changes the checksum, triggering re-resolution
- **Simplicity**: No need to handle path normalization across platforms
- **Size**: Lockfiles remain small (patches can be large)

The cost is that the lockfile doesn't show patch file provenance (only a checksum).
This is consistent with path dependencies,
which don't track paths or even checksums in the lockfile.
Both are treated as local to the project.

This can only be changed via lockfile version bump once stabilized.

↩ [*Lockfile format*][lockfile-format]

### Why apply patches before resolution?
[rationale-timing]: #why-apply-patches-before-resolution

**Chosen**: Before resolution (same as existing `[patch]` behavior).
Patches are registered first, then the registry returns patched summaries when queried.
See [When patches are applied](#when-patches-are-applied) for the flow.

**Alternatives considered**:

1. **On demand during resolution**: Complex because `[patch]` entries can be added or removed
   depending on application of other `[patch]`, requiring complicated backtracking.

2. **After resolution, re-resolve if needed** (explored in [PR #14055]):
   Re-resolve if any summary changed due to patching. This would allow patching `package.version`
   to work correctly, since re-resolution would pick up the new version.
   However, this adds complexity (loop detection, convergence guarantees).

Both alternatives would introduce dual behavior: patch-files would have different timing
than existing `[patch]` entries (git, path), making the feature harder to understand.

**Rationale for before resolution**:
- Matches existing `[patch]` semantics exactly
- Simpler implementation, no re-resolution loops
- Patches to `Cargo.toml` dependencies still work (patched summary is returned before resolution)

**Consequences of this timing decision**:

1. Patching `package.name` or `package.version` is an error:
   Cargo requires the patched crate to preserve the original package identity.
   If a patch changed the version from `1.0.0` to `2.0.0`,
   nothing in the dependency graph would require `foo 2.0.0`,
   and the patched crate would go unused or cause resolver confusion.

2. Version-range patches not supported: Patches apply to a specific source
   (registry + version, or git + rev). No version-range matching like
   `foo = { version = ">=0.3.35, <0.3.37", patches = [...] }`.

   The [`time` crate compatibility issue][PR #14452] illustrates why version-range patches would be valuable:
   multiple versions (0.3.35, 0.3.36) were affected,
   requiring separate `[patch]` entries per locked version.
   However, with exact versions we know which source to patch before resolution.
   With ranges, we'd need the "after resolution, re-resolve" approach we explicitly avoided.

   Additional concerns with version-range patches:
   - Patch compatibility: A patch for `foo-0.3.35` may not apply cleanly to `foo-0.3.36`
   - Lockfile semantics: Current lockfiles pin exact versions;
     ranges would create ambiguity about which patch was actually applied

   Version-range patches are deferred ([?][future-version-range]).

[PR #14452]: https://github.com/rust-lang/cargo/pull/14452

This **cannot be changed** after stabilization.
While version-range syntax itself would be additive,
implementing it properly requires "after resolution" timing,
conflicting with the timing we're stabilizing here.
Adding it later would require either inconsistent
timing behavior within `[patch]`, or a new mechanism outside the current design.

↩ [*When patches are applied*][when-patches-are-applied], [*Guide-level explanation*][guide-level-explanation]

### Why different path roots for registry vs git?
[rationale-path-roots]: #why-different-path-roots-for-registry-vs-git

**Chosen**: Paths inside patch files are relative to package root for registry dependencies,
repository root for git dependencies.

**Alternative**: Always use package root for both.

**Rationale**: Git patches are typically generated with `git diff`,
which outputs paths relative to the repository root.
Requiring users to edit these paths would add friction.
Registry packages don't have this context, so package root is the natural choice.
Both `git diff` and GNU `diff -u` produce paths with `a/` and `b/` prefixes,
so Cargo implicitly strips one leading component (like `patch -p1`)
to arrive at the actual file path relative to the appropriate root.

**Caveat**: This parallel behavior may surprise users who expect consistency.
However, it matches how patches are typically generated for each source type.

This **cannot be changed** after stabilization.

↩ [*Patch file format*][patch-file-format]

### Why unified diff with strict matching?
[rationale-patch-format]: #why-unified-diff-with-strict-matching

**Chosen**: Unified diff and git diff formats
(output of GNU `diff -u`, `git diff`, or `git format-patch`)
with built-in strict matching (no fuzz).

**Alternatives**:
- Other diff formats (context diff, ed script, etc.)
- Configurable `[patchtool]` for external tools (explored in PR #13779)

**Rationale for unified diff**: The most widely used format:
- Understood by virtually all version control and code review tools
- Output of `git diff`, GNU `diff -u`, and GitHub's `.diff` URLs
- 40+ years of tooling support

**Caveat**: There is no official standard for unified diff. Variants exist
(GNU diff, git diff, etc.) with minor differences.
The built-in parser is best-effort and handles common variants.

**Rationale for built-in application**:
- Zero configuration: Works out of the box on all platforms
- Consistency: Same behavior everywhere (no `patch` vs `gpatch` differences)
- Security: No arbitrary command execution
- Portability: No dependency on external tools (important for Windows)

**Why no fuzz factor**: Tools like GNU `patch` support `--fuzz=N` to allow context lines
to differ slightly. We intentionally do not support fuzz because:
- Safety: Fuzzy matching can apply patches to the wrong location silently
- Reproducibility: The same patch might apply differently on different machines
- Explicit failures are better: If context has drifted, the patch should fail loudly

See [*What about fuzz factor support?*][future-fuzz] in Future Possibilities.

**Why no partial patch application**: When a patch has multiple hunks and one fails,
we fail the entire patch rather than applying successful hunks. Rationale:
- Correctness first: A partially-applied patch often leaves code in a broken state
- Predictability: Users can reason about "patch applied" vs "patch not applied",
  not "patch partially applied in unknown ways"
- Matches git behavior: `git apply` fails entirely if any hunk fails

**Format support**: The implementation supports unified diff and git diff formats.
Mode changes (executable bit) are not supported ([?][future-extended-format]).
Starting strict on unsupported features allows loosening later
without breaking existing users.

This is **safe to change** after stabilization
(e.g., adding `[patchtool]` configuration, fuzz support, or extended format support).

↩ [*Patch file format*][patch-file-format], [*Guide-level explanation*][guide-level-explanation]

### Why require identical patches for git dependencies from the same repo?
[rationale-identical-git-patches]: #why-require-identical-patches-for-git-dependencies-from-the-same-repo

**Chosen**: All packages from the same git repository must use identical patches.

```toml
# ERROR: different patches on same git repo
[patch.crates-io]
bar = { git = "https://example.com/repo", patches = ["a.patch"] }
baz = { git = "https://example.com/repo", patches = ["b.patch"] }
```

**Alternative**: Allow different patches per package within the same git repo.

**Rationale**: Git repositories are fetched as a whole unit.
Cargo preserves the workspace structure when fetching git dependencies.
Allowing different patches would require:
- Multiple copies of the same repo with different patches, or
- Complex patch merging logic

The current design keeps git source handling simple and predictable.

This is **safe to change** after stabilization (could relax to allow different patches,
though UX and performance implications would need resolution).

↩ [*Guide-level explanation*][guide-level-explanation]

### Why not support path dependencies?
[rationale-no-path-deps]: #why-not-support-path-dependencies

**Chosen**: The `patches` field is rejected for path dependencies.

**Alternative**: Allow patches on path dependencies.

**Rationale**: Path dependencies point to local, editable source.
Users can modify the files directly, so patches add unnecessary indirection.

Additionally, the semantics of patching local files are awkward: would Cargo copy every source file?
What are the boundaries if a path refers to `../../../` or other unrelated locations?

If the path points to a read-only or generated location,
users should copy it to an editable location or use a different source type.

This is **safe to change** after stabilization (could add support later).

↩ [*Syntax*][syntax], [*Guide-level explanation*][guide-level-explanation]

## Prior Art

This feature has a long history of community interest and implementation attempts.

### Rust ecosystem: Tracking Issue (#4648, 2017)

The original feature request ([rust-lang/cargo#4648]) was opened in October 2017,
motivated by integration with large build systems like Buildroot and OpenWrt.
The issue has accumulated 110+ thumbs-up reactions, indicating strong community demand.

Key points from the discussion:
- Patch files are standard practice in Linux distribution packaging
- Storing entire forks for small fixes is unwieldy at scale
- The feature should integrate with existing `[patch]` semantics

[rust-lang/cargo#4648]: https://github.com/rust-lang/cargo/issues/4648
[RFC 3177]: https://github.com/rust-lang/rfcs/pull/3177
[PR #13779]: https://github.com/rust-lang/cargo/pull/13779
[PR #14055]: https://github.com/rust-lang/cargo/pull/14055

### Rust ecosystem: RFC 3177 (2021)

[RFC 3177] proposed a `patchfiles` field with version-range support:

```toml
[patch.crates-io]
bar = { version = "2.0", patchfiles = ["patches/bar.patch"] }
```

Key design decisions in that RFC:
- Patches applied after download, before compilation
- `version` field required when no `path`/`git` specified
- Recursive re-resolution if patches modify `Cargo.toml`
- Unified diff format via Rust implementation (not external tools)

The RFC was not merged, partly due to unresolved questions about
dependency graph updates when patches modify `Cargo.toml`.

[RFC 3177]: https://github.com/rust-lang/rfcs/pull/3177

### Rust ecosystem: Experimental PRs (#13779 and #14055, 2024)

Two experimental implementations explored different timing strategies:

**[PR #13779]** (patch-before-resolution):
- Required exact version (`=1.0.0`) to know what to download before resolution
- Patches could modify dependencies, affecting resolution
- Used external `[patchtool]` configuration
- Downside: Exact version pinning is not ergonomic

**[PR #14055]** (patch-after-resolution, re-resolve if needed):
- Applied patches after initial resolution
- Re-resolved only if patched `Cargo.toml` changed dependencies
- More flexible version requirements
- Downside: Re-resolution adds complexity

Both PRs were closed but informed the current design.
The current design takes inspiration from both approaches:
patches are applied before resolution (like #13779)
and the patched summary is returned to the resolver,
so dependency changes in `Cargo.toml` take effect
without the re-resolution loop of #14055.

### Rust ecosystem: Cargo Team Guidance (2024)

In team discussions, an early direction considered
preventing patches from touching `Cargo.toml` to simplify the feature.
However, because patches are applied before resolution
and the patched summary is returned to the resolver,
patching `Cargo.toml` to modify dependencies works naturally
without re-resolution loops.
The one restriction is that patching `package.name` or `package.version` is an error,
since the patched crate must preserve the original package identity.
See [*Why apply patches before resolution?*][rationale-timing]
for the detailed design and its consequences.

### JavaScript package managers (Yarn, pnpm, Bun)

[Yarn Berry][yarn-patch], [pnpm][pnpm-patch], and [Bun][bun-patch] all support patch files
with similar patterns:

**Shared workflow**: `<tool> patch <pkg>` extracts package to temp directory for editing,
then `<tool> patch-commit` generates the patch file.

**Declaration**: pnpm and Bun use `patchedDependencies` mapping package specs to patch paths:

```yaml
# pnpm-workspace.yaml (pnpm) or package.json (Bun)
patchedDependencies:
  express@4.18.1: patches/express@4.18.1.patch
  foo@^2.0.0: patches/foo-2.patch      # version range (pnpm only)
  bar: patches/bar.patch               # all versions (pnpm only)
```

Yarn uses a `patch:` protocol embedding patch info in package references:
```
"lodash@patch:lodash@npm%3A4.17.21#./patches/lodash.patch::version=4.17.21&hash=abc123"
```

**Lockfile format**: All store relative paths + hashes in the lockfile. This means:
- Lockfiles contain machine-specific paths (portability concern)
- Patch files must exist at the declared paths for lockfile to be valid

This differs from Cargo's approach of storing only checksums in the lockfile.

**Key limitation shared by all JS tools**: Patches applied at fetch time, not resolution time.
This means patches **cannot alter dependencies**.
Cargo differs here: patches are applied before resolution
and the patched summary is returned to the resolver,
so dependency changes in patched `Cargo.toml` files take effect.

**Notable differences**:
- pnpm supports version-range matching (exact > range > name-only priority)
- Bun uses copy-on-write optimization for global cache sharing

### Nix

[Nix][nix-patches] (functional package manager) supports patches via the `patches` attribute on derivations:

```nix
stdenv.mkDerivation {
  name = "foo-1.0";
  src = fetchurl { url = "..."; };
  patches = [
    ./fix-1.patch
    ./fix-2.patch
  ];
}
```

**Workflow**: Patches are listed declaratively; applied during `unpackPhase`.
Multiple patches are applied in order (patch series).

**Key feature**: Patches are first-class in the derivation language,
making them inspectable and composable across package definitions.

**Lockfile equivalent**: Nix's lock file (`flake.lock`) records sources but not patch content;
patches live in the source tree.

### quilt

[quilt] is the Unix/Linux standard for managing patch series:

```bash
$ quilt new fix-bug.patch
$ quilt edit src/main.rs
$ quilt refresh
$ ls patches/
00-fix-bug.patch
01-another-fix.patch
```

**Workflow**: Create patches incrementally, edit files, `quilt refresh` updates patches.
`series` file lists patches in order.

**Key feature**: Designed for sequential, ordered patch application.
Widely used in Debian and other Linux distributions.

**Storage**: Patches stored in `patches/` directory with a `series` file listing order.

### Debian (dpkg-source)

[Debian packages][dpkg-source] use `debian/patches/` with a `series` file:

```
# debian/patches/series
01-fix-segfault.patch
02-add-feature.patch
03-security.patch
```

**Workflow**: Patches applied automatically during `dpkg-source -x`.
Developers use `quilt` to manage the patch series.

**Key feature**: Patches are part of the source package (`.dsc`);
multiple patches applied in defined order.

**Portability**: Series file is human-readable and version-controlled;
applied consistently across all systems.

### Comparison

| Aspect | JS (Yarn/pnpm/Bun) | Nix | quilt / Debian | Cargo |
|--------|-------------------|-----|----------------|-------|
| Lockfile content | Path + hash | N/A (derivation) | series file | Checksum only |
| Version matching | Per-descriptor¹ | N/A | N/A | Per-source |
| Can alter deps? | ❌ No | ❌ No | ❌ No | ✅ Yes² |
| Patch storage | `patches/` dir | User location | `patches/` + `series` | User-defined |
| Patch series? | ❌ Single per pkg | ✅ Yes | ✅ Yes (ordered) | ✅ Yes (array) |
| Lockfile portable? | ⚠️ Relative paths | ✅ Yes | ✅ VCS-relative | ✅ Checksum only |

¹ pnpm supports version ranges; Yarn/Bun require exact versions.

² Patching `Cargo.toml` can modify dependencies, add features, etc.
The patched metadata is used by the resolver before resolution.
However, patching `package.name` or `package.version` is an error
(see [*Why apply patches before resolution?*][rationale-timing]).

**Cargo's differentiator**: Lockfile stores only checksums, not paths.
This prioritizes portability (works across machines with different directory structures)
at the cost of requiring manifest + patch files to reconstruct the full picture.

**Cargo vs. system-level tools**: Unlike quilt/Debian (which apply patches during extraction)
or Nix (which applies at build time),
Cargo applies patches to a cached copy of the fetched source,
integrating with Cargo's existing caching and build model.


## Unresolved Questions

### During implementation (before stabilization)

- Cache directory structure The layout under `$CARGO_HOME/patched-src/`. The RFC specifies
  that patched sources are cached with checksum-based invalidation, but the exact path structure
  is an implementation detail.

- Patch application error messages What context to show when a patch fails to apply
  (full diff, failing hunk only, suggestions for resolution).


## Future Possibilities

### What about `[patchtool]` configuration?

[↩][patch-file-format] External patch tools for advanced formats or stricter application:
```toml
# .cargo/config.toml
[patchtool]
path = "/usr/bin/patch"
args = ["-p1", "--fuzz=0"]
```

The built-in application is best-effort and does not support fuzz matching or binary patches.
This would allow users to opt into GNU patch, quilt, or custom tooling for advanced formats.
[Bazel provides similar `patch_tool` configuration][Bazel].

[Bazel]: https://bazel.build/rules/lib/repo/http#http_archive-patches

### What about a `cargo patch` subcommand?

[↩][guide-level-explanation] A helper to extract dependencies, make changes, generate patch files and `[patch]` entries.

### What about version-range patches?

[↩][when-patches-are-applied] Apply one patch to multiple versions:

```toml
[patch.crates-io]
foo = { version = ">=0.3.35, <0.3.37", patches = ["fix.patch"] }
```

This would be useful for security or compatibility fixes affecting multiple versions
(e.g., the [`time` crate issue][PR #14452]).

See [Patch application timing](#patch-application-timing) for why this requires
"after resolution" timing incompatible with current design.

### What about extended patch formats?

[↩][patch-file-format] The initial implementation supports basic unified diff.
Future extensions could include:

- Binary patches: `git format-patch` can encode binary file changes
- File renames/copies: `git diff --find-renames` detects moved files
- Executable bit and mode changes
- Non-UTF-8 encoding

These can be added incrementally without breaking existing patches.

### What about fuzz factor support?

[↩][patch-file-format] If strict matching proves too rigid in practice,
per-patch opt-in fuzz could be added:

```toml
[patch.crates-io]
foo = { version = "=1.0.0", patches = ["fix.patch"], fuzz = 1 }
```

See [*Why unified diff with strict matching?*][rationale-patch-format]
for why strict is the default.
Changing the default in either direction is a behavior change:

* strict-to-fuzzy risks silent misapplication,
* fuzzy-to-strict breaks previously-applying patches.

Per-patch opt-in avoids this tension by keeping strict matching as the permanent default.

### Conditional patches

Some patches only make sense on specific platforms.
Possible directions include:

- **Per-patch target gate:**
  `patches = [{ file = "win.patch", target = "cfg(windows)" }]`
  scoped to the patch-files feature itself.
- **Broader `[patch]` target support:**
  `[target.'cfg(windows)'.patch.crates-io]`
  extending the existing `[target]` table
  to conditionally apply entire `[patch]` sections.

The current design does not close the door on any of these.
In the meantime,
users can write patches that account for all target platforms
by using conditional compilation within the patched source.

### Same-source substitution

Replacing a package with a *different* package from the same source
(e.g., `tokio-tar` with `astral-tokio-tar` from crates.io) is a separate feature request.
See [rust-lang/cargo#9227](https://github.com/rust-lang/cargo/issues/9227).

When the two packages have small divergence,
patch-files can serve as a workaround.
For example, diff the two package sources and apply the result as a patch.
However, this does not address the general case of swapping package identity within the dependency graph,
and produces unmaintainable patches when the packages diverge significantly.

--------------------------------------------------------------------------------

Notes:

* cargo install `.cratev2.json`
* PackageId
* PackageIdSpec
* SourceId
* `cargo vendor` for patch with patch files?

<!-- Section anchor definitions for backlinks -->

[guide-level-explanation]: #guide-level-explanation
[syntax]: #syntax
[patch-file-format]: #patch-file-format
[lockfile-format]: #lockfile-format
[when-patches-are-applied]: #when-patches-are-applied

<!-- Future possibility anchor definitions for [?] links -->

[future-patchtool]: #what-about-patchtool-configuration
[future-cargo-patch]: #what-about-a-cargo-patch-subcommand
[future-version-range]: #what-about-version-range-patches
[future-extended-format]: #what-about-extended-patch-formats
[future-fuzz]: #what-about-fuzz-factor-support
