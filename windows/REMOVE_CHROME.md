# Remove Google Chrome

`remove_chrome.yml` uninstalls Google Chrome from Windows hosts and cleans up everything it
leaves behind — the Google updater services and scheduled tasks, the program directories,
per-user profile data, and Chrome's registry keys.

Tested against Windows Server with Chrome 154.0.8037.98 (standalone EXE install).

## Usage

```bash
ansible-navigator run -m stdout remove_chrome.yml -e _hosts=windows
```

`_hosts` is required — the play targets `hosts: "{{ _hosts }}"`, matching `install_iis_aap.yml`
and `remove_iis.yml` so the host pattern can come from an AAP survey or `-e` at launch.

**Dry run** (non-destructive; discovery and reporting still execute):

```bash
ansible-navigator run -m stdout remove_chrome.yml -e _hosts=windows --check
```

## Variables

| Variable | Default | Purpose |
|---|---|---|
| `_hosts` | *(required)* | Host pattern to target. |
| `chrome_remove_user_data` | `true` | Delete `AppData\Local\Google` under every profile in `C:\Users`. **Irreversible** — bookmarks, saved passwords, extensions. |
| `chrome_remove_updater` | `true` | Remove `Google*` / `gupdate*` services, `Google*` scheduled tasks, and the updater directories. Prevents Chrome silently reinstalling itself. |
| `chrome_remove_policies` | `false` | Also delete `SOFTWARE\Policies\Google\Chrome`. Off by default — these are normally GPO-managed and will simply return on the next policy refresh. |
| `chrome_registry_keys` | *(list)* | HKLM-relative Chrome keys to remove. |
| `chrome_policy_keys` | *(list)* | HKLM-relative policy keys, used only when `chrome_remove_policies` is true. |

## How it works

Chrome ships three ways — enterprise MSI, standalone EXE, and Chocolatey — and the MSI product
GUID changes with **every release**, so a hardcoded `product_id` goes stale immediately. Instead
the playbook discovers the install at run time:

1. **Find Chrome uninstall entries** — enumerates the HKLM uninstall hives (64-bit and
   `WOW6432Node`) plus `HKEY_USERS\*` for per-user installs, filtering `DisplayName -like
   'Google Chrome*'`. Splits `UninstallString` into executable + arguments and flags `is_msi`.
2. **Stop Chrome and updater processes** — Chrome's uninstaller fails or hangs if the browser
   is still running.
3. **Uninstall** — the discovered entries are split with `selectattr`/`rejectattr` into two
   `ansible.windows.win_package` tasks, one for MSI (`product_id`) and one for EXE
   (`path` + `--uninstall --system-level --force-uninstall`).
4. **Clean up** — updater services and scheduled tasks, program directories, per-user data, and
   registry keys. Each destructive step discovers its targets first and then deletes with a real
   module (`win_service`, `win_file`) rather than a shell one-liner.
5. **Verify** — re-runs the same discovery and asserts nothing matches. This is what makes the
   playbook safe to wire into a compliance job.

The discovery script is defined once as `chrome_find_script` and reused by both the find and the
verify tasks.

## Things worth knowing

### You must use `ansible-navigator`, not `ansible-playbook`

`pywinrm` is installed in the execution environment, not on the control node. Running
`ansible-playbook` directly fails with:

```
winrm or requests is not installed: No module named 'winrm'
```

### `win_regedit` cannot delete these keys

On a 64-bit host, `win_regedit` (and `Remove-Item`) resolve the **parent** `SOFTWARE\Google`
through the WOW64-redirected view, where `Chrome` genuinely does not exist — that view holds
only `Google\Update`. The result is a misleading error for a key that is plainly present:

```
Cannot delete a subkey tree because the subkey does not exist.
```

Worse, via `Remove-Item` the error is **non-terminating**, so the task reports `changed` while
deleting nothing at all. The playbook therefore opens the `Registry64` base key explicitly, which
addresses the real key and makes the `WOW6432Node` paths literal rather than redirected. This is
also why `chrome_registry_keys` are HKLM-relative (`SOFTWARE\...`) rather than `HKLM:\...`.

### Part of `SOFTWARE\Google\Chrome` will survive, and that is correct

Windows plants **Microsoft-owned** native messaging host registrations under Chrome's key:

- `NativeMessagingHosts\com.microsoft.browsercore` — Windows Security BrowserCore
- `NativeMessagingHosts\com.microsoft.defender.browser_extension.native_message_host` — Defender

These are protected. Even `reg delete /f` returns `ERROR: Delete request is partially completed`
(rc 1), and Defender recreates them regardless. Registry cleanup is deliberately **best-effort**:
anything that cannot be deleted is reported rather than failing the play. A normal run prints:

```
TASK [Report registry keys left in place]
"msg": "Protected keys retained (owned by Windows, not Chrome): SOFTWARE\\Google\\Chrome"
```

That message is expected and not an error.

### Chrome's `setup.exe` return codes

`expected_return_code` is `[0, 19, 3010]` — Chrome uses installer-specific codes and `19` means
"uninstall successful". If you hit a different non-zero code, read it before widening the list.

### `product_id` for EXE installs is not a GUID

For the EXE install the uninstall registry key is literally named `Google Chrome`, and that is
what gets passed as `product_id`. `win_package` uses it to check installed state, so it works —
it just does not look like the GUID you would expect from an MSI.

### Per-user installs in unloaded hives

The `HKEY_USERS\*` sweep only sees profiles whose hive is **currently loaded**, so a per-user
Chrome belonging to a logged-off account will not appear in the uninstall list, and the final
assert will not flag it. The `chrome_remove_user_data` sweep deletes its files regardless.
Acceptable on a server; worth knowing on a multi-user box.

### Linting and `--syntax-check`

Both fail with `'_hosts' is undefined` unless you pass it:

```bash
ansible-navigator run remove_chrome.yml --syntax-check -e _hosts=windows -m stdout
```

`ansible-lint` has no `--extra-vars`, so to lint the rest of the rules, lint a copy with the host
resolved:

```bash
mkdir -p /tmp/cl
sed 's|hosts: "{{ _hosts }}"|hosts: windows|' remove_chrome.yml > /tmp/cl/remove_chrome.yml
ansible-lint --project-dir /tmp/cl /tmp/cl/remove_chrome.yml   # passes at the 'production' profile
```

This is not specific to this playbook — `remove_iis.yml` behaves the same way. It is a side
effect of the `_hosts` convention.

### Check mode

The discovery tasks carry `check_mode: false` so they still run under `--check` and report what
would happen. The final assert is skipped in check mode (`when: not ansible_check_mode`), since
Chrome is obviously still installed at that point.

## Verification

A clean run ends `failed=0` with `changed=0` on the second pass:

```
TASK [Assert Chrome is removed]
"msg": "Google Chrome is not installed"

PLAY RECAP
<host> : ok=12  changed=0  unreachable=0  failed=0  skipped=6
```

To confirm independently on the target:

```powershell
Get-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*',
                 'HKLM:\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*' |
  Where-Object DisplayName -like 'Google*'
Get-Service 'Google*','gupdate*'
Get-ScheduledTask | Where-Object TaskName -like 'Google*'
Test-Path 'C:\Program Files\Google\Chrome'
```

All four should come back empty / `False`.

## Requirements

Only `ansible.windows`, already pinned in `requirements.yml` (`>=2.0.0`) and present in the EE.
No new collections, and no `community.windows` dependency.
