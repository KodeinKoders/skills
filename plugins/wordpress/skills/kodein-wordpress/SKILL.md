---
name: kodein-wordpress
description: Run authenticated requests against a WordPress site's REST API (/wp-json/). Use when the user asks to work on a WordPress site or domain — read, create, update or delete posts, pages, media, categories, tags, menus, users or settings, "publish this on the blog", "update the page on wp-example.com" — or gives a WordPress URL to act on. Looks up credentials in a local .wordpress-agent.conf in the working directory, else in the per-domain file in ~/.config/wordpress-agent/ (%LOCALAPPDATA%\wordpress-agent\ on Windows), before calling the API. Not for editing a WordPress theme or plugin's PHP source.
---

# WordPress REST API

When asked to work on a WordPress site, talk to its REST API (`https://<domain>/wp-json/wp/v2/...`).
Look for credentials first; if there are some, every request must carry them.

## 1. Find the credentials

The domain is the host of the site (`wp-example.com`, no scheme, no path, no port unless the site uses one).

1. Check whether a **local** `.wordpress-agent.conf` exists in the current working directory.
   If it does, use it **instead of** the global file: do not read the global one, not even to fill in missing keys.
2. Otherwise, check whether the **global** conf file for the domain exists (see the table below), e.g. `wp-example.com.conf`.
3. If neither exists, proceed **without** authentication (public endpoints only) and tell the user that no credentials were found, naming the paths you looked at.
4. If one exists, read it as a **HOCON** file and resolve `auth.username` and `auth.password`.

The local file is not named after a domain: it applies to whatever site you are asked to work on from that directory.
Only the current working directory counts; do not search parent directories.
It holds the same keys as the global file, so everything below applies to both.

| OS | File |
| --- | --- |
| macOS, Linux | `${HOME}/.config/wordpress-agent/<domain>.conf` |
| Windows | `%LOCALAPPDATA%\wordpress-agent\<domain>.conf` (PowerShell: `$env:LOCALAPPDATA\wordpress-agent\<domain>.conf`; Git Bash: `${LOCALAPPDATA}/wordpress-agent/<domain>.conf`) |
| WSL | `${HOME}/.config/wordpress-agent/<domain>.conf`, as on Linux |

On Windows, the directory is `wordpress-agent` directly under `LOCALAPPDATA` (no `.config` folder), and `HOME` is usually not set.
Backslashes or forward slashes both work.
If unsure of the OS, check both locations.

The file is HOCON, not JSON or properties: those two keys may be written in any valid HOCON form (nested object, dotted path, `=` or `:`, quoted or unquoted, comments, substitutions, includes…).
Never assume one fixed layout; see [references/hocon.md](references/hocon.md) for the forms to expect.

Typical content:

```hocon
auth {
    username = editor@wp-example.com
    password = "xxxx xxxx xxxx xxxx"
}
```

If both `auth.username` and `auth.password` resolve, authenticate **every** request with them (HTTP basic auth).
If either is missing or cannot be resolved, make unauthenticated requests and tell the user why.

The password is normally a WordPress *application password*; spaces in it are part of the value, keep them.

## 2. Make authenticated requests

Use `curl` with basic auth.
Keep the credentials out of the command line (it is visible in process listings) and out of your output:
feed them to curl through a config on stdin.

```sh
curl -sS -K - "https://wp-example.com/wp-json/wp/v2/posts?per_page=5" <<EOF
user = "${WP_USER}:${WP_PASS}"
EOF
```

Set `WP_USER` / `WP_PASS` in the same shell command from the values you read (do not `export` them across commands, do not echo them).
Escape `\` and `"` in a value placed inside the curl config.

On Windows in PowerShell, use `curl.exe` (plain `curl` is an alias of `Invoke-WebRequest`) and pipe the config to stdin:

```powershell
@"
user = "$($WpUser):$($WpPass)"
"@ | curl.exe -sS -K - "https://wp-example.com/wp-json/wp/v2/posts?per_page=5"
```

The `sh` examples below use single-quoted JSON bodies, which `cmd.exe` and PowerShell do not handle the same way: on Windows, write the body to a temporary file, pass `-d "@file.json"`, then delete the file.

Useful patterns:

```sh
# create (JSON body; status defaults to draft unless "publish" is given)
curl -sS -K - -X POST "https://wp-example.com/wp-json/wp/v2/posts" \
  -H "Content-Type: application/json" \
  -d '{"title":"Hello","content":"<p>Body</p>","status":"draft"}' <<EOF
user = "${WP_USER}:${WP_PASS}"
EOF

# update
curl -sS -K - -X POST "https://wp-example.com/wp-json/wp/v2/posts/42" -H "Content-Type: application/json" -d '{"title":"New title"}' <<EOF
user = "${WP_USER}:${WP_PASS}"
EOF

# who am I (verifies the credentials)
curl -sS -K - "https://wp-example.com/wp-json/wp/v2/users/me?context=edit" <<EOF
user = "${WP_USER}:${WP_PASS}"
EOF
```

## Rules

- **Never print, log, commit or paste the password** — not in commands shown to the user, not in answers, not in files in the project.
  When quoting a command, show the placeholders, not the values.
- Never write credentials anywhere but their conf file, and never edit that file unless asked.
  If a local `.wordpress-agent.conf` is inside a git repository and is not ignored (`git check-ignore .wordpress-agent.conf`), warn the user to add it to `.gitignore`, and never stage or commit it.
- Use HTTPS. Do not disable certificate checks (`-k`) unless the user says so.
- When unsure which routes exist, discover them with `GET https://<domain>/wp-json/` (lists namespaces and routes, including custom post types and plugin routes).
- Reads first: for anything destructive (`DELETE`, `?force=true`, overwriting content, changing users or settings, publishing), state what you are about to do and get confirmation unless the user already asked for exactly that.
- Create content as `draft` unless the user asks to publish.
- Paginate list results (`per_page` ≤ 100, `page`; the totals are in the `X-WP-Total` / `X-WP-TotalPages` response headers).
- On `401`/`403`: report it, check that the credentials in the conf file match the domain, and do not retry in a loop.
  On `404` for `/wp-json/`, try `https://<domain>/?rest_route=/` (plain permalinks).
- Content fields take HTML (Gutenberg block markup if the site uses it); fetch an existing item with `?context=edit` and mirror its `content.raw` format when editing.
