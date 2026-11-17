---
name: deploy-nextcloud-windows
description: Deploy and maintain a private Nextcloud All-in-One file sync server on Windows using Docker Desktop with WSL 2 and Tailscale Serve. Use for hands-on setup, sync troubleshooting, account isolation, shared folders, and ownership migration on this stack.
---

# Deploy Nextcloud AIO on Windows

Use the [repository deployment guide](../../../README.md) for the maintained PowerShell commands and UI steps. If this skill was copied out of the repository, use the [public deployment guide](https://github.com/Ao0O0oZ/nextcloud-windows-tailscale#readme). Check the linked official documentation when an installed version behaves differently.

Use Tailscale policy to control access to the private HTTPS endpoint. Use Nextcloud accounts and grants to control access to files after login. Test both when diagnosing a permission problem.

## Inspect the Windows host

Run the inspection on the intended server. If you only have a terminal on another machine, give the user the commands to run locally and continue from their results. Use PowerShell on Windows for Docker Desktop and Tailscale operations.

```powershell
Get-Volume | Select-Object DriveLetter, FileSystem, HealthStatus, SizeRemaining
wsl --version
wsl --list --verbose
Get-Command docker, tailscale -ErrorAction SilentlyContinue
Get-NetTCPConnection -State Listen -LocalPort 8080,11000 -ErrorAction SilentlyContinue
```

If Docker is available, check `docker --context desktop-linux info --format '{{.OSType}}'`, then list AIO containers and volumes using the same context. Treat an unavailable daemon as an unknown state. If Tailscale is available, check `tailscale status` and `tailscale serve status`. Keep peer inventories and real device names out of shared artifacts. Ask for or infer only the paths and access scope needed for the requested deployment.

- **Existing AIO state.** Preserve named volumes and populated directories. Inspect the live mastercontainer, existing mounts and port bindings. Start an intact stopped deployment through its existing startup procedure. For reset or migration, determine where the Borg backup and its key are stored and follow [AIO recovery guidance](https://github.com/nextcloud/all-in-one#backup). A Windows data directory alone cannot restore the database or accounts.
- **Fresh host.** Select an empty directory on an internal NTFS disk for server data. Use a separate sync directory for each desktop client, including any client on the server. The same disk can host both directories. Creating or resizing partitions, deleting state, and restoring over an existing instance require authorization for those operations.

## Bring up a fresh instance

1. Install or update WSL and reboot if Windows requests it. Install and start Docker Desktop using its WSL 2 Linux engine. Check that `docker-desktop` shows WSL version 2 and that the engine reports `linux`. Docker Desktop supplies its own WSL environment. A separate Ubuntu installation is optional.
2. Have the user sign in to Tailscale locally and enable MagicDNS and HTTPS Certificates in the intended tailnet. Help scope access to the intended clients on TCP 443. If the user needs a neutral device name, set it before issuing certificates because the full name appears in certificate transparency logs. Use the actual DNS name shown by Serve for this deployment.
3. Check Serve before modifying it. Reuse the existing private mapping if it already matches the deployment. For a free 443 mapping, use `tailscale serve --bg --https=443 http://127.0.0.1:11000` in the Windows admin shell and confirm `tailnet only`. Resolve any conflicting mapping with the user before replacing it. Keep access private by using Serve and Windows loopback bindings.
4. Use the guide's guarded **启动 Nextcloud AIO** PowerShell block. Change the data directory to the chosen path. Keep the AIO mastercontainer name and named configuration volume, use the documented Windows Docker socket mount, and bind the management UI and Apache to `127.0.0.1`. Translate `X:\path` to `/run/desktop/mnt/host/x/path` for `NEXTCLOUD_DATADIR`. Set the data directory before initial setup. For this private proxy setup the example uses `SKIP_DOMAIN_VALIDATION=true`, which skips AIO's precheck. Verify the private HTTPS endpoint with normal certificate validation after startup.
5. Open the AIO wizard on the server at `https://127.0.0.1:8080`. Its management certificate is self-signed. Have the user complete any browser trust prompt and use the management passphrase locally. In AIO enter the Serve hostname without the scheme, then start the containers. Wait for the application to become ready. Ask the user to sign in and create a regular Nextcloud account for syncing when needed. Keep passwords, auth keys, and recovery keys in their intended local input or credential store.
6. On the requested clients, connect Tailscale, open the HTTPS URL, and use Nextcloud Desktop's browser authorization flow. Select an empty sync directory for a new connection. If it already contains files, check the existing mapping and contents before connecting it to a fresh or restored server. For Windows recycle-bin behavior, configure virtual files and mark the sync root always available locally. After quitting Nextcloud Desktop, set `moveToTrash=true` in the existing `[General]` section of `%APPDATA%\Nextcloud\nextcloud.cfg`, preserve other keys, and restart the client.

Continue the authorized steps yourself when you can operate the target host. For login, access consent, or a reboot that needs the user, state the exact action and resume from a new status check afterward. When a command fails, inspect the current state before rerunning it. Use bounded waits for startup and check the relevant component's recent logs if it stays unhealthy.

## Manage accounts, shared folders, and migrations

For existing accounts, follow [the account isolation and migration section](../../../README.md#6-多账号隔离与迁移).

- Use regular accounts for daily sync and a separate administrator account. Keep personal directories under their owners' accounts. Grant access to common folders through Team Folders and the intended Nextcloud user group. Check group membership and existing shares on personal folders and their contents, then verify each account can open its own files and only the shared files intended for it. Use separate browser sessions for these checks.
- Use folder selection or ignore rules to reduce what a client syncs. Enforce account isolation with server-side grants. Put Git worktrees and automation scratch directories outside all sync roots. If a directory goes missing, pause the affected sync connections and check recent server-side move activity and the trash before changing files.
- Pause relevant sync clients and verify a recovery path before transferring ownership. Select only the intended source folder. If the transfer fails, inspect both storage locations, the file index, share records, and server logs before retrying. If the files are verified at the target and only its index is missing, preserve old versions and scan that destination with `occ files:scan --path=/USER/files/PATH`. Verify the recovered files and review the old shares and version history separately.
- For retirement, verify a working administrator login, migrate or disconnect old sync connections, and check the old account's trash, versions, and shares. Disable the account for a reversible cutover. Delete the account or a migration archive only once the user has authorized the deletion and any history they want to keep is preserved.
- If Android photo uploads are requested, configure a custom local folder and cloud destination in Auto Upload. Test with a newly created photo and one moved into the folder. For a request for two-way photo sync, verify each direction and deletion behavior before reporting it complete.

## Verify the requested outcome

- **Service.** Check the active AIO containers, the actual persistent mount at `/mnt/ncdata`, and the management and Apache port bindings. Request `/status.php` over the private HTTPS address and require `installed=true`, `maintenance=false`, and `needsDbUpgrade=false`. Check that the login page loads. An unauthenticated WebDAV `401` shows that the route requires authentication. A signed-in file test verifies sync.
- **File sync.** With the requested client connected, create a uniquely named test file, confirm it reaches the server, modify it from the other side, and compare the content after sync. Test from another authorized device on a separate network when available. Use test files for deletion, recovery, and offline checks.
- **Recycle bin.** Once the test file is fully downloaded, delete it through the server and check the Windows Recycle Bin. Restore it and confirm the upload. The Nextcloud web trash is a separate recovery route. Report the Windows test result for that client and configuration.
- **Startup.** If reboot verification is requested, check after Windows sign-in that Docker Desktop, AIO, Tailscale Serve, and the desktop client recover. Login startup is distinct from availability before any Windows user signs in.

Report the verified URLs, data and sync paths, and test outcomes to the owner. Mark remote checks, restart tests, or recovery tests as pending when the required device or user interaction was unavailable.
