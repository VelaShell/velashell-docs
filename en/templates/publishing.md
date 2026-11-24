# Building, Packaging, Signing and Publishing

> See also: [Development Guide](dev-guide.md) · [CLI Manual](../cli/cli.md) · [SDK Reference](../sdk/sdk-reference.md)
> Plugin marketplace: <http://market.easilynet.top>

This page covers the road from "it runs on my machine" to "other people can install it".
Three things: **produce a correct package**, **sign it under a stable identity**, and
**get it to users**.

---

## 1. Decide these before you publish

### 1.1 The plugin id is immutable

Once published, never change `id` in `plugin.json`. It is simultaneously:

- the prefix of every command id (`acme.snippets.run`);
- the namespace of the plugin's private storage and secrets (changing it makes user data vanish);
- the key for upgrades (same id overwrites).

Convention: `<publisher>.<name>`, charset `[a-z0-9.-]`, must start and end with a letter or
digit, ≤64 characters.

### 1.2 Version and the three compatibility gates

| Field | Judged by | Consequence of getting it wrong |
| --- | --- | --- |
| `version` | semver, `1.2.0` / `1.2.0-beta.1` | The marketplace and upgrade logic order by it |
| `apiLevel` | Higher than the host supports → refused | Only bump on **breaking** changes; normally stays `1` |
| `minHostVersion` | Older host → Incompatible | Use the first compatible host release that provides a newer host API the plugin uses; never put an Avalonia version or `minSdkVersion` here |
| `minSdkVersion` | Host's bundled SDK older → Incompatible | **Required for plugins that use newer SDK surface** |

`apiLevel` is too coarse: it only covers breaking changes, and added interface methods or DTO
fields are not breaking. A plugin that uses `IRemoteTunnelApi` (SDK 1.2), `ITerminalViewApi`
(1.3) or workspace variants (1.3.1) without declaring `minSdkVersion` will install and activate
on an older host and then throw `MissingMethodException` on the first call. Declared, the old
host marks it Incompatible **during discovery** and says exactly what to upgrade.

> Host-side toolchain surface (such as `HostRegistry` in SDK 1.4) needs no declaration —
> plugin code never calls it.

`vela-plugin doctor` checks all three gates against the host on your machine.

### 1.3 Host mode

- `inProcess` (default): a collectible ALC inside the host process. Panels dock as main-window
  tabs, and the `Protocols` / `Workspaces` / `RemoteTunnel` / `Network` / `TerminalView` capabilities are
  available.
- `isolated`: its own `VelaShell.PluginHost` process; crashes and hangs cannot take the host
  down. Panels are standalone card windows.

Changing `hostMode` after publishing is a **behavioural change**: treat it as a minor version
bump and say so in the release notes.

---

## 2. Release build

```bash
dotnet build -c Release
```

A plugin project references exactly one package, `VelaShell.PluginSdk.Build`, which brings:

- `EnableDynamicLoading=true` → generates `deps.json` (how the ALC resolves the plugin's own
  dependencies) and copies dependencies to the output;
- `plugin.json` copied into the output directory (the host discovers plugins by that file);
- **shared assemblies excluded**: `VelaShell.PluginSdk.dll` and `Avalonia*.dll` never enter the
  output or `.vpx` — the loader forces them to fall back to the host's copies (types must be
  identical across ALCs), so shipping them only bloats the package;
- for plugin **direct references** to `Avalonia` / `Avalonia.AvaloniaEdit`, a check that resolved
  package versions match `VelaAvaloniaVersion` / `VelaAvaloniaEditVersion`;
- post-build manifest validation with the same rules the host applies.

`.Build` and the SDK do not depend on or transmit Avalonia. A plugin using AXAML/UI must read the exact
versions from the target host's `VelaShell/src/Directory.Packages.props`, set the corresponding properties,
and add explicit `PackageReference ... ExcludeAssets="runtime"` items; direct references provide the
AXAML/compiler build and compile assets. If those packages are not used, declare no such properties. A package
whose name starts with `Avalonia` but is not provided by the host is still treated as shared by the loader and
cannot bypass that rule by being bundled.

Native dependencies (P/Invoke `.so` / `.dylib` / `.dll`) resolve through the RID assets in
`deps.json`; ship one package per target platform, or include every RID's native assets.

---

## 3. Packing `.vpx`

```bash
dotnet build -c Release -t:PackVpx
# → bin/vpx/acme.snippets-0.1.0.vpx

vela-plugin pack bin/Release/net11.0 -o dist/
```

`.vpx` is a dedicated container, not a renamed zip:

```text
┌ 64-byte header ──────────────────────────────────────────┐
│ magic 56 50 58 1A · format version · flags · payload len │
│ payload SHA-256 · mask nonce · header CRC32              │
└──────────────────────────────────────────────────────────┘
  masked zip payload
  optional: trailing signature block (JSON: alg / publicKey / signature)
```

The signature covers those 64 bytes, and the header contains the payload length and digest —
equivalent to signing the whole package.

The package should contain: the entry DLL, `deps.json`, the plugin's own third-party
dependencies, `plugin.json` and resources. It should **not** contain
`VelaShell.PluginSdk.dll`, `Avalonia*.dll`, or any key or credential.

---

## 4. Signing

### 4.1 Create a key (once)

```bash
vela-plugin keygen -o ~/keys/acme.pem
```

ECDSA P-256 + SHA-256 (not Ed25519: it is not in the BCL, and the contract assembly is not
allowed to take heavyweight third-party dependencies).

> **The private key is your publisher identity.** Back it up offline, never commit it. Losing it
> means a new key, and a new key means every existing user is asked to confirm trust again.

### 4.2 Sign and verify

```bash
dotnet build -c Release -t:PackVpx -p:VelaSigningKey=$HOME/keys/acme.pem
vela-plugin sign   dist/acme.snippets-0.1.0.vpx -k ~/keys/acme.pem
vela-plugin verify dist/acme.snippets-0.1.0.vpx -k "MFkwEw..."
```

### 4.3 What users see

| Package state | Install experience |
| --- | --- |
| Valid signature, key already trusted here | Installs directly |
| Valid signature, publisher not trusted | Shows the **public-key fingerprint** and asks the user to verify it through your official channel, then "trust publisher and install" |
| Unsigned | Yellow warning ("plugins can execute code with your account permissions; install only if you trust the source"), explicit confirmation required |
| Broken signature / tampered content | **Always refused**, with no override |

After a successful install the host records a **protected installation receipt** (content hash
with integrity protection). If the plugin's files change afterwards, it is marked Invalid at
startup and the user is told to reinstall — this defends against post-install tampering.

So: **sign your packages and publish your fingerprint** (README, marketplace page, website).
The fingerprint is the only thing users can verify, and upgrade continuity depends on it.

---

## 5. Publishing to the marketplace

The VelaShell plugin marketplace is at <http://market.easilynet.top>.

Once listed, users have two ways to install your plugin:

```bash
vela-plugin install <your plugin id>       # command line, since SDK 1.5.0
```

Or download the `.vpx` from the marketplace page and pick it in **Plugin manager → Install .vpx…**.
Both land in the same directory. The single difference: the manager also records a **protected
installation receipt** for post-install tamper detection, which the CLI cannot produce (the key
lives inside the host process) - in exchange the CLI verifies the file digest, the container
digest, the signature, manifest consistency and host compatibility before writing anything.
See the [CLI manual](../cli/cli.md#2-installing-from-the-marketplace).

> The exact submission form/API belongs to the site; the list below is the material you need
> regardless of its shape.

### 5.1 Submission checklist

| Material | Source | Notes |
| --- | --- | --- |
| The `.vpx` package | `dotnet build -c Release -t:PackVpx` | Must be a **signed** Release build |
| id / version / display name | `plugin.json` | Must match the manifest inside the package exactly |
| Publisher and author | `plugin.json` `publisher` / `author` | The manager page shows `author`, falling back to `publisher` |
| Public-key fingerprint | output of `keygen` / `sign` | The users' only identity proof; **do not change it after the first submission** |
| Description and screenshots | yours | One sentence on the problem it solves, at least one screenshot |
| Compatibility | `apiLevel` / `minHostVersion` / `minSdkVersion` | Lets the marketplace tell users whether their VelaShell can install it |
| Release notes | yours | One paragraph per version; always mention new capabilities or behaviour changes |
| License and source URL | `plugin.json` `license` / `homepage` | Strongly recommended for open-source plugins |

Before submitting:

```bash
vela-plugin doctor
vela-plugin verify dist/xxx.vpx
vela-plugin info   dist/xxx.vpx
```

### 5.2 Cadence and updates

- **Same-id upgrade** replaces the directory and keeps plugin data (KV / secrets / time series).
  Migrate your own data structures — only uninstall clears data.
- **Always use the same private key for the same id**, or users face the trust prompt again.
- **Downgrade**: users can install an older `.vpx`; make sure the older version tolerates data
  written by the newer one (rebuild what it cannot read).
- **Withdrawal**: unlist the affected version and ship a fix quickly. The host has **no** remote
  revocation mechanism, so versions already on user machines do not disappear — which is exactly
  why compatibility and safety belong before the release, not after.

### 5.3 Producing packages in CI (GitHub Actions)

```yaml
name: release-plugin
on:
  push:
    tags: ['v*']

jobs:
  pack:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with: { dotnet-version: '11.0.x' }

      - name: Restore signing key
        run: |
          install -m 600 /dev/null "$RUNNER_TEMP/key.pem"
          printf '%s' "${{ secrets.VELA_PLUGIN_KEY }}" > "$RUNNER_TEMP/key.pem"

      - name: Pack
        run: dotnet build -c Release -t:PackVpx -p:VelaSigningKey="$RUNNER_TEMP/key.pem"

      - name: Verify
        run: |
          dotnet tool install -g VelaShell.Plugin.Cli
          vela-plugin verify bin/vpx/*.vpx -k "${{ vars.VELA_PLUGIN_PUBKEY }}"

      - uses: actions/upload-artifact@v4
        with:
          name: vpx
          path: bin/vpx/*.vpx
```

### 5.4 Publishing to the marketplace straight from CI (trusted publishing)

If you don't want to upload by hand on every release, let GitHub Actions push the `.vpx` to the
marketplace directly. This uses **trusted publishing** (the same mechanism as PyPI's Trusted Publishing):
CI holds **no marketplace secret at all**. It asks GitHub for an OIDC token that is valid only for this run
and expires within minutes, and the marketplace checks that the repository, workflow and environment in
that token match what you registered. The package then goes through quarantine, signature checks and the
virus scan exactly like a manual upload.

**One-time setup:**

1. **Upload the first version by hand.** Trusted publishing only publishes new versions of an existing plugin;
   it never claims a new id — who owns a plugin id must be decided by a signed-in person.
2. On the marketplace's "My plugins" page, click **Trusted publishing** for the plugin and register:
   - **GitHub repository**: `owner/repo`. The marketplace looks up the repository's numeric id on GitHub and
     matches on that id from then on — renaming the repository is fine, while someone recreating a deleted
     repository under the same name will not match. Private repositories can only be registered if the
     marketplace is configured with a GitHub token.
   - **Workflow file**: the file name under `.github/workflows/`, e.g. `release.yml` (case-sensitive).
   - **Environment** (optional, **strongly recommended**): e.g. `market`. Without it, anyone with write access
     to the repository can publish by pushing a branch that edits this workflow. With it, go to the repository's
     Settings → Environments and add required reviewers, or allow only release tags to deploy to it —
     the rule must match your tag format: `[0-9]*.[0-9]*.[0-9]*` for tags like `1.2.3`, `v*` only for `v1.2.3`.
     A wrong rule raises no error; it just blocks every release at the environment.

   If one repository publishes several plugins, tick the others under "Also register for".

**Add a job to the workflow** (after the `pack` job from 5.3):

```yaml
  publish-market:
    needs: pack
    runs-on: ubuntu-latest
    environment: market          # must match the registered environment; drop it if you registered none
    permissions:
      id-token: write            # required to request the OIDC token
      contents: read
    steps:
      - uses: actions/download-artifact@v4
        with: { name: vpx, path: dist }

      - name: Publish to VelaShell market
        shell: bash
        run: |
          for vpx in dist/*.vpx; do
            token=$(curl -sSf -H "Authorization: Bearer $ACTIONS_ID_TOKEN_REQUEST_TOKEN" \
              "$ACTIONS_ID_TOKEN_REQUEST_URL&audience=velashell-market" | jq -r .value)
            status=$(curl -sS -o resp.json -w '%{http_code}' -H "Authorization: Bearer $token" \
              -F "file=@$vpx" -F "releaseNotes=Release notes (Markdown, optional)" \
              https://market.easilynet.top/api/uploads/trusted)
            case "$status" in
              202) echo "$vpx is in quarantine; it publishes automatically once the checks pass" ;;
              409) echo "$vpx: this version is already published, skipping" ;;
              *)   cat resp.json; exit 1 ;;
            esac
          done
```

| Status | Meaning |
| --- | --- |
| `202` | In quarantine; published automatically once the checks pass. See the result under "My uploads" |
| `409` | This version is already published. This is what a rerun of the same tag gets; treat it as success |
| `403` | No trusted publisher matches the token: check the repository, the workflow file name (case-sensitive) and the environment |
| `404` | The plugin does not exist on the marketplace yet: upload the first version by hand |
| `401` | Invalid token — usually the `audience` is not `velashell-market`, or the job lacks `id-token: write` |

The authoritative `audience` is the one shown in the marketplace's "Trusted publishing" drawer (a self-hosted
marketplace may have changed it). The first-party plugin repository velashell-plugins uses exactly this; see
the `publish-market` job in its `.github/workflows/release.yml`.

---

## 6. Pre-release checklist

- [ ] `id` unchanged, `version` bumped
- [ ] `apiLevel` / `minHostVersion` / `minSdkVersion` match the APIs actually used
- [ ] `displayName` / `description` / `author` / `license` / `homepage` filled in
- [ ] Release build, `vela-plugin doctor` reports no blocking problem
- [ ] No `VelaShell.PluginSdk.dll` / `Avalonia*.dll` / keys / credentials in the package
- [ ] Signed, and `vela-plugin verify -k <public key>` passes
- [ ] Installed once on a clean machine or clean data root and the main flow works
      (`vela-plugin dev run --data-root ~/.velashell-clean` builds such an environment quickly)
- [ ] Release notes describe new capabilities and behaviour changes
