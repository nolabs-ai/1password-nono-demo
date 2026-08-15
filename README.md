# nono + 1Password GitHub CLI Demo

This demo runs GitHub CLI (`gh`) and, optionally, Claude Code inside a
[nono](https://github.com/always-further/nono) sandbox. nono loads a GitHub
token from 1Password, keeps the real credential outside the sandbox, and gives
the sandboxed process a phantom `GH_TOKEN`. The proxy replaces that phantom
with the real token only for GitHub API requests permitted by the profile.

The included Demonator script walks through the complete flow: loading the
1Password item, making permitted read requests, blocking writes at both the
command and HTTP layers, and showing that the same controls apply to `gh` calls
made by an agent.

Although we use the GitHub CLI in this demonstration, the same credential-brokering
approach can be applied to any command-line tool that uses an environment variable
for authentication. The profile can be adapted to other tools by changing the
`command_policies.commands.<tool>` section and the credential route's `endpoint_policy` 
to match the tool's API.

## What this demonstrates

- The GitHub token is resolved from an `op://` reference before the sandbox is
  applied. `op:://` is a dedicated 1Password CLI URL scheme that nono recognizes.
- Sandboxed `gh` sees a nono phantom token, never the real token.
- `gh issue view`, `gh issue list`, and selected `gh api` reads are allowed.
- Issue comments, issue creation, issue closure, release uploads, and other
  mutations are denied.
- Command policy still applies when `gh` is launched by a shell or an agent.
- L7 endpoint policy checks the HTTP method and path, so an allowed `gh api`
  command cannot use `POST` to bypass the read-only policy.
- A profile without a credential route reaches GitHub anonymously, providing a
  useful comparison with the brokered request.

## How credential brokering works

```text
1Password                     nono                         sandboxed gh
    |                           |                               |
    |-- real GitHub token ----->|                               |
    |                           |-- phantom GH_TOKEN ---------->|
    |                           |<-- request + phantom token ----|
    |                           |-- check HTTP policy            |
    |                           |-- inject real token ----------> GitHub API
```

The real token remains in nono's proxy. It is not written to a temporary file,
placed in the child process environment, or returned by `gh auth token` inside
the sandbox. A request must pass both policy layers:

1. The `gh` argument policy must allow the command.
2. The credential route's endpoint policy must allow the resulting HTTP method
   and path.

## Files

- [`ghcli-profile-1password.json`](./ghcli-profile-1password.json) is the main
  macOS profile. It extends the `nolabs-ai/claude` base profile and adds a
  read-only, 1Password-backed `gh` command policy.
- [`ghcli-1password-demo.yml`](./ghcli-1password-demo.yml) is the scripted
  Demonator presentation.
- [`basic.json`](./basic.json) is the comparison profile. It permits `gh` to
  run but grants no credential, so GitHub API requests are anonymous.

## Prerequisites

This demo is currently configured for macOS with Homebrew paths. Install:

- nono
- Demonator
- 1Password CLI (`op`) with access to the desktop app or an authenticated
  account
- GitHub CLI (`gh`)
- `jq`
- Claude Code, only for the final two agent demonstrations

This can be adapted to linux, but the profile must be edited to match the local
env vars and filesystem paths.

For example:

```bash
cargo install nono-cli # or brew install nono
cargo install demonator
brew install 1password-cli gh jq
```

Confirm the required commands are available:

```bash
command -v nono demonator op gh jq
op account list
gh auth status
```

## 1. Store the GitHub token in 1Password

The supplied profile uses this 1Password secret reference:

```text
op://Development/GitHub CLI Token/password
```

It expects a Password item named `GitHub CLI Token`, with a `password` field,
in the `Development` vault. To seed that item from an existing GitHub CLI
login, run the following once:

```bash
gh auth token \
  | jq -Rs '{
      title: "GitHub CLI Token",
      category: "PASSWORD",
      fields: [{
        id: "password",
        type: "CONCEALED",
        purpose: "PASSWORD",
        label: "password",
        value: sub("\\n$"; "")
      }]
    }' \
  | op item create --vault=Development -
```

This pipeline sends the token over standard input; it does not put the token in
command arguments or a temporary file. If the item already exists, do not
create a duplicate. Instead, update it in 1Password or change the profile to
refer to the existing item.

If your vault, item, or field has a different name, edit this property in
`ghcli-profile-1password.json`:

```text
command_policies.credentials.github-1password.credential_key
```

Verify that `op` can resolve the reference without printing the secret:

```bash
op read 'op://Development/GitHub CLI Token/password' >/dev/null \
  && echo '1Password credential is available'
```

## 2. Check the local `gh` path

The profile pins both the `gh` executable and its Homebrew Cellar directory.
Homebrew changes the versioned Cellar path when `gh` is upgraded. Compare the
installed path with the values in the profile:

```bash
realpath "$(command -v gh)"
jq '{
  executable: .command_policies.commands.gh.executable,
  sandbox_reads: .command_policies.commands.gh.from.session.sandbox.fs_read
}' ghcli-profile-1password.json
```

Update every versioned `/opt/homebrew/Cellar/gh/<version>/bin` entry in
`ghcli-profile-1password.json` if it does not match the installed version. The
unversioned executable should normally remain `/opt/homebrew/bin/gh`.

The included profile currently contains `gh/2.89.0`, but it's likely
that a newer version is available and already on your machine. If so,
update the profile to match the installed version. The profile will not
work if the pinned Cellar path is stale.

## 3. Validate the profile and demo

From this directory, run:

```bash
nono profile validate ./ghcli-profile-1password.json
demonator --dry-run -c ./ghcli-1password-demo.yml
```

The first command validates the profile and its referenced groups. The second
parses the scripted presentation and prints each step without executing it.

## 4. Run the scripted demo

Start the interactive presentation from this directory:

```bash
demonator -c ./ghcli-1password-demo.yml
```

Demonator pauses before each step. Press Enter to continue. The script shows,
in order:

1. The command, credential, filesystem, argument, and endpoint policy.
2. A direct `op read` check with the token output discarded.
3. nono loading the same `op://` reference and giving `gh` a phantom token.
4. An authenticated API read through the brokered route.
5. The same read with `basic.json`, where no credential is granted.
6. Allowed high-level and low-level issue reads.
7. A comment rejected by the `gh` argument policy.
8. A `POST` rejected by the L7 endpoint policy.
9. A shell-wrapped mutation rejected at the child-tool boundary.
10. Optional Claude Code read and write attempts under the same policy.

Denied commands return a non-zero status. That is the expected result for the
write demonstrations, not a demo failure.

The Claude commands use `--dangerously-skip-permissions` to remove Claude's own
interactive approval prompts. This does not disable nono: the sandbox and its
`gh` command policy remain the enforcement boundary. If Claude Code is not
installed or authenticated, stop after the shell-wrapping demonstration or
remove the final two steps from the YAML file.

## Run the core checks manually

Set the profile path once:

```bash
export ONEPASSWORD_PROFILE=./ghcli-profile-1password.json
```

Show that `gh auth token` receives a phantom value. The hexadecimal value in
this output is deliberately not the GitHub token:

```bash
nono run -vv --allow-cwd \
  --profile "$ONEPASSWORD_PROFILE" \
  -- gh auth token
```

Make an authenticated request allowed by the endpoint policy:

```bash
nono run -s --allow-cwd \
  --profile "$ONEPASSWORD_PROFILE" \
  -- gh api rate_limit | jq '{core: .resources.core}'
```

Read an issue using a high-level command:

```bash
nono run -s --allow-cwd \
  --profile "$ONEPASSWORD_PROFILE" \
  -- gh issue view 1052 \
    --repo nolabs-ai/nono \
    --json title,url,state | jq
```

Read the same issue through the constrained API command:

```bash
nono run -s --allow-cwd \
  --profile "$ONEPASSWORD_PROFILE" \
  -- gh api repos/nolabs-ai/nono/issues/1052 --jq .title
```

Try a mutation that the argument policy denies before `gh` contacts GitHub:

```bash
nono run -s --no-diagnostics --allow-cwd \
  --profile "$ONEPASSWORD_PROFILE" \
  -- gh issue comment 1052 \
    --repo nolabs-ai/nono \
    --body 'nono argument-policy test'
```

Try a mutation through the otherwise allowed `gh api` prefix. The endpoint
policy denies the `POST` before it reaches GitHub:

```bash
nono run -s --no-diagnostics --allow-cwd \
  --profile "$ONEPASSWORD_PROFILE" \
  -- gh api -X POST \
    repos/nolabs-ai/nono/issues/1052/comments \
    -f body='nono endpoint-policy test'
```

## Adapting the demo

To use another repository or issue, update both policy and demo data. In
particular, change the repository paths under:

```text
command_policies.commands.gh.from.session.sandbox.credentials[0].endpoint_policy
```

Then update the `--repo` arguments and `repos/<owner>/<repo>/...` API paths in
`ghcli-1password-demo.yml`. Changing only the demonstration commands will not
work because the endpoint policy defaults to deny.

For a different 1Password item, update only the credential route's
`credential_key`. Keep the credential binding name (`github-1password`)
consistent between `command_policies.credentials` and the command sandbox.

## Troubleshooting

### `gh` is denied a file under `/opt/homebrew/Cellar/gh/...`

The pinned Cellar path is stale. Follow **Check the local `gh` path** above and
replace the old version in every relevant filesystem list in the profile.

### The profile works but an API request is denied

This normally means the request does not match an endpoint allow rule. Inspect
the effective allow and deny lists:

```bash
jq '.command_policies.commands.gh.from.session.sandbox.credentials[0].endpoint_policy' \
  ghcli-profile-1password.json
```

The endpoint policy has `"default": "deny"`, so every required method and path
must be explicitly allowed.

### `gh` prompts or reports that it is unauthenticated

The profile sets `GH_PROMPT_DISABLED=1` and injects `GH_TOKEN`. Confirm that:

- `op read` succeeds outside nono.
- the command is using `ghcli-profile-1password.json`, not `basic.json`.
- the `github-1password` credential binding is still present on the `gh`
  command sandbox.
- the request targets `https://api.github.com`, matching the credential route's
  upstream.

### TLS certificate errors

The profile enables trusted TLS interception so nono can enforce HTTP endpoint
policy. If `gh` reports a certificate error, validate that the profile retains
`network.tls_intercept.ca_lifecycle: "trusted"` and use a nono build that
supports macOS proxy CA trust.

## Security notes

- Do not replace the `op://` source with a literal token in the profile.
- Do not print the result of `op read` while presenting or recording the demo.
- Treat the 1Password item as a real GitHub credential and scope it as narrowly
  as the demonstration permits.
- The phantom token protects the credential from the child process; command and
  endpoint policies determine what the child can do with the brokered access.
- Keep the endpoint policy default-deny when adapting the profile.
