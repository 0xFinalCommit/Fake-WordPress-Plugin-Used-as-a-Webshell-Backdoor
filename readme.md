# Case Study: Fake WordPress Plugin Used as a Webshell Backdoor

**Site:** blog.blueheavenweb.com (WordPress 6.9.4, own site)
**Service:** Incident response, malware analysis, cleanup and hardening
**By:** Blue Heaven Web (blueheavenweb.com)

---

## Summary

An attacker logged in to WordPress, **uploaded a fake plugin called "Zedd Plugins"**, and used it to hide a set of **PHP webshells and remote loaders**. The plugin itself looked harmless, but its hidden `m/` folder let the attacker upload files, edit files and run code from their own server. The same backdoor was then dropped into **every site on the same hosting account**, and `wp-config.php` was edited to lock the real owner out of plugin and file management.

| | |
|---|---|
| **Malware type** | Webshell / backdoor kit disguised as a plugin |
| **Disguise** | Plugin named "Zedd Plugins" with an Indonesian description, no malicious code in the main file |
| **Source** | Single IP from Indonesia |
| **Reach** | 5 sites in one hosting account |

---

## What We Found

### 1. A plugin that was not installed by the owner

The plugins list showed an unknown **Zedd Plugins** entry (inactive, but its files were live on disk).

![Unknown plugin in WordPress admin](\Screenshot 2026-04-17 233254.jpg)

In the file manager, the plugin folder contained a harmless-looking main file and a hidden folder `m/`, created minutes earlier.



### 2. The real payload was in the hidden folder

The main plugin file only adds a dummy menu page. The `m/` folder held 8 PHP files with these capabilities:

- **Remote loaders:** tiny files that download code from the attacker's server and run it, so the attacker can change the payload at any time
- **File uploader:** lets the attacker upload any file
- **File manager / overwriter:** scans, edits, deletes and changes permissions of files
- **Staged downloader:** pulls extra tools onto the server

Because the plugin was inactive, it did not matter: the files can be called directly by URL.

### 3. Spread across the hosting account

The same loader file (`CDX7.php`) was found in the web root of **five sites** on the same account, plus a copy inside the fake plugin. The attacker reached other sites through the shared hosting account, not through separate WordPress logins.



Fresh unknown files also appeared in the site root, next to normal files.



### 4. wp-config.php was tampered with

The attacker added a "Secured by..." comment and set `DISALLOW_FILE_EDIT` and `DISALLOW_FILE_MODS` to `true`. This blocks the owner from installing, updating or deleting plugins from the dashboard, which **protects the attacker's plugin from being removed**. (Database details in this file are hidden in the screenshot.)



### 5. The access log shows the exact attack sequence

One IP (Indonesia) shows this order of events:

1. Opens `wp-login.php`, then a successful login to `/wp-admin/`
2. Opens **Plugins > Add New > Upload**
3. Posts to `update.php?action=upload-plugin` (plugin upload)
4. Hours later, direct requests and POSTs to files inside `/plugins/zeddsiti/m/`



---

## Attack Timeline

| Step | Action |
|---|---|
| 1 | Attacker logs in to WordPress admin (valid credentials) |
| 2 | Uploads the fake plugin through the dashboard |
| 3 | Uses webshells to drop loaders into other sites on the account |
| 4 | Edits `wp-config.php` to block plugin/file changes |
| 5 | Returns later to use the uploader and file manager |

**Likely entry point:** a valid admin login (weak, reused or leaked password). The logs do not show a password guess attack in the visible window, so credentials should be treated as compromised.

---

## Cleanup

1. Backup everything, and keep a copy of the malicious files for evidence.
2. Delete the `zeddsiti` plugin folder, including `m/`.
3. Delete the dropped loader files from the root of **all five sites**, and any unknown files next to them.
4. Remove the injected lines from `wp-config.php`.
5. Reinstall WordPress core and compare plugins/themes with clean originals.
6. **Rotate everything:** WordPress admin passwords, hosting password, FTP, database user and password, and WordPress security keys. Remove unknown users.
7. Block the attacker IP and the malware's domain at server level.
8. Enable two-factor authentication and limit login attempts.
9. Scan every site on the account and keep off-site backups.

---

## Lessons

- **Fake plugins are a favourite persistence trick.** Always review the plugin list and the plugins folder for names you don't recognise.
- **Inactive does not mean safe.** Files in a plugin folder can still be called directly.
- **One hacked site on shared hosting can mean many.** Scan every site in the account.
- **Check wp-config.php.** A sudden `DISALLOW_FILE_MODS` can be a lockout, not hardening.
- **Access logs tell the story:** login, plugin upload, then direct calls to the dropped files.

---
