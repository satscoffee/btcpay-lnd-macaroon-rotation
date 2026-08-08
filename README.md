# Rotate LND Macaroons on a LunaNode BTCPay Server

> ## ⚠️ READ THIS FIRST
>
> **If you are here because of the August 2026 BTCPay Server advisory, update to 2.4.2
> before doing anything else.**
>
> Rotating macaroons does **not** remediate an unpatched vulnerability. If an attacker
> retains access, they may be able to obtain newly generated credentials again.
>
> **Correct order:**
>
> 1. **Update to BTCPay Server 2.4.2 or later**, or take the server offline until you can
> 2. **Check for unauthorized access** — unfamiliar user accounts, unrecognized API keys,
>    changed 2FA settings, attacker-configured payout or pull-payment destinations,
>    unexpected wallet transactions
> 3. **Only then** rotate credentials using this guide
>
> **Also update NBXplorer to 2.6.10.** The official release notes recommend this alongside
> the BTCPay update. Don't stop at BTCPay itself.
>
> If you run Core Lightning or another Lightning backend, refresh its authentication
> strings too — this guide covers LND only.
>
> Verify your version in the BTCPay footer, or:
>
> ```bash
> sudo docker inspect generated_btcpayserver_1 --format='{{.Config.Image}}'
> ```
>
> ### What the vulnerability was
>
> Per the [official v2.4.2 release notes](https://github.com/btcpayserver/btcpayserver/releases/tag/v2.4.2):
>
> > *"This release contains fix of a critical vulnerability that is being actively
> > exploited. You need to update as fast as you can."*
>
> The security fix listed is **"Fix TOTP two-factor authentication bypass via Greenfield
> Basic authentication."** Reported by `@brunoerg` and `@benthecarman` of the Bitcoin Red
> Team.
>
> **This tells you what to check for.** The attack path was authentication bypass against
> the Greenfield API — so prioritize looking for unauthorized account and API activity, not
> just wallet transactions.
>
> **Note the breaking change:** 2.4.2 disables Greenfield Basic authentication by default
> five minutes after account creation. If an integration of yours relied on Basic auth, it
> will stop working — API key authentication is the supported path.
>
> ### Verify this yourself
>
> **Do not take any version number in this guide on faith.** Advisories change and this
> document is a snapshot. Check current guidance before acting:
>
> - [v2.4.2 release notes](https://github.com/btcpayserver/btcpayserver/releases/tag/v2.4.2) — primary source
> - [BTCPay Server releases](https://github.com/btcpayserver/btcpayserver/releases)
> - [BTCPay Server on X](https://x.com/BtcpayServer)
> - [BTCPay Server chat](https://chat.btcpayserver.org/)
>
> Independent coverage:
> [The Block](https://www.theblock.co/news/ecosystems/2026-08-07-btcpay-warns-actively-exploited-vulnerability-could-drain-funds-411170) ·
> [Decrypt](https://decrypt.co/375159/bitcoin-payment-service-btcpay-critical-flaw-active-attack) ·
> [crypto.news](https://crypto.news/btcpay-server-warns-active-exploit-may-drain-funds/)
>
> *Note: the v2.4.2 release notes do not themselves mention macaroon rotation. Guidance to
> replace macaroons and recreate `macaroons.db` comes from BTCPay's advisory communications
> as relayed in press coverage. Rotation is sound practice after a suspected credential
> exposure regardless — but patch first.*

> ### Scope: this is maintenance, not incident response
>
> This guide covers **planned, voluntary credential rotation** on a server you believe is
> healthy.
>
> **If you have reason to believe the server was actually compromised, stop here.** Take the
> affected server offline and follow current BTCPay Server security and incident-response
> guidance. **Do not rely on macaroon rotation alone.**

This guide walks you through rotating the LND macaroon credentials on a BTCPay Server
running on LunaNode.

It is written for users who are comfortable logging into their server with SSH and copying
commands, but who do not need to be Linux or Lightning developers.

The SSH instructions are LunaNode-specific. The remaining steps apply to
`btcpayserver-docker` LND deployments that match the configuration checks in Step 3.

**The procedure:**

- invalidates the existing LND macaroon credentials
- creates a new macaroon root key
- regenerates the standard LND macaroon files
- regenerates the restricted LND subserver macaroons used by the tested BTCPay configuration
- verifies that the credentials actually changed
- verifies that your Lightning channel inventory is accounted for afterward

It does not intentionally modify your wallet, seed, channel database, channels, or TLS
certificate.

---

## Scope

This guide was tested on:

- LunaNode Ubuntu VPS
- BTCPay Server installed with `btcpayserver-docker`
- Bitcoin mainnet
- LND `v0.21.0-beta`
- LND container `btcpayserver_lnd_bitcoin`
- standard Bolt database backend

This guide deliberately verifies those assumptions before deleting anything.

**If your configuration does not match the checks below, stop. Do not modify the commands
to make your server fit the guide.**

This guide is for LND, not Core Lightning.

---

## What is a macaroon?

An LND macaroon is a bearer credential that authorizes an application to communicate with
your Lightning node.

BTCPay uses macaroons internally. Apps such as RTL, Zeus, Zap, ThunderHub, and custom API
integrations may use them too.

This procedure replaces LND's macaroon root-key database. As a result, **every macaroon
issued under the previous root key becomes invalid**, including custom macaroons that may
be stored somewhere other than the standard locations used in this guide.

Any external app using an old macaroon must be reconnected or given a newly generated
credential afterward.

---

## Before you begin

Do this at a quiet time when you are not expecting payments.

Your LND container will restart and Lightning peers will disconnect temporarily. Peer
connections may take several minutes to return.

The commands in this guide do not issue a channel-close command. However, **a Lightning
peer can independently close a channel at any time**, including during this maintenance
window. That is why the guide records your channel state before and after the rotation.

### You must have your LND seed

In BTCPay: **Server Settings → Services → LND Seed Backup**

Record the seed offline if you have not already done so. **Do not continue without it.**

### Important Lightning backup warning

Do not treat `channel.db` like a normal restorable backup.

Do not restore an old Lightning data directory or casually roll the VPS back to an old
snapshot. A stale Lightning channel database can contain obsolete commitment state and can
put channel funds at risk.

For recovery from stale or uncertain channel-state data, use the LND seed and Static
Channel Backup rather than restoring old channel state.

---

## Step 1 — SSH into LunaNode

Open Terminal on macOS/Linux, or PowerShell / Windows Terminal on Windows.

LunaNode Ubuntu installations normally use the username shown under your VM's **Initial
Login Details** in the LunaNode panel.

```bash
ssh ubuntu@YOUR_SERVER_IP
```

If LunaNode shows a different username, use that instead.

The cursor will not move while you type your password. That is normal.

After logging in, you should see a command prompt on the server.

---

## Step 2 — Become root and set the working variables

Copy and paste:

```bash
sudo su -

export LND=btcpayserver_lnd_bitcoin
export NET=mainnet
export M=/data/data/chain/bitcoin/$NET
export WORK=/root/lnd-macaroon-rotation-$(date +%F-%H%M%S)

mkdir -p "$WORK"

echo "Working directory: $WORK"
```

**These variables are only valid for the current SSH session.** If you disconnect and
reconnect, run this block again before continuing.

Verify that the expected LND container exists:

```bash
docker ps --format '{{.Names}}' | grep -x "$LND"
```

Expected:

```text
btcpayserver_lnd_bitcoin
```

If nothing appears, **stop.** You may be using Core Lightning, a different container name,
or a nonstandard installation.

If you use RTL, record its container name now for Step 14:

```bash
export RTL=$(docker ps --format '{{.Names}}' | grep -i rtl)
echo "RTL container: $RTL"
```

---

## Step 3 — Verify that this guide matches your LND installation

Check the LND version:

```bash
cd ~/btcpayserver-docker
./bitcoin-lncli.sh getinfo | head -6
```

This guide was tested on LND `v0.21.0-beta`.

Inspect the configured macaroon paths and database backend:

```bash
docker exec "$LND" grep -E \
'^(adminmacaroonpath|invoicemacaroonpath|readonlymacaroonpath|db\.backend)=' \
/data/lnd.conf
```

For the three macaroon paths, you should see:

```text
adminmacaroonpath=/data/admin.macaroon
invoicemacaroonpath=/data/invoice.macaroon
readonlymacaroonpath=/data/readonly.macaroon
```

For `db.backend`:

- **no `db.backend` line** is expected on the standard configuration, because LND defaults
  to Bolt
- `db.backend=bolt` is also acceptable
- if you see `sqlite`, `postgres`, or `etcd`, **stop**

This guide assumes the Bolt macaroon database stored as `macaroons.db`.

Now inspect the actual files:

```bash
echo "=== /data ==="
docker exec "$LND" ls -la /data/

echo
echo "=== mainnet directory ==="
docker exec "$LND" ls -la "$M/"
```

For the tested configuration, `/data/` contains:

```text
admin.macaroon
invoice.macaroon
readonly.macaroon
```

The mainnet directory contains:

```text
chainnotifier.macaroon
invoices.macaroon
router.macaroon
signer.macaroon
walletkit.macaroon
macaroons.db
```

It should also contain important files that this guide will not delete, including:

```text
wallet.db
walletunlock.json
channel.backup
```

If your macaroon layout differs, or you see additional custom `.macaroon` files, **stop and
identify what uses them before continuing.** Replacing `macaroons.db` will invalidate custom
macaroons too, even though this guide does not delete their files.

---

## Step 4 — Verify the node is healthy before touching anything

```bash
cd ~/btcpayserver-docker
./bitcoin-lncli.sh getinfo | tee "$WORK/getinfo-before.txt"
```

Confirm:

```text
"synced_to_chain": true
```

Record the number of active channels, inactive channels, and peers.

Check for channels that are already opening or closing:

```bash
./bitcoin-lncli.sh pendingchannels | tee "$WORK/pending-before.txt"
```

For the cleanest maintenance window you want `"total_limbo_balance": "0"` and empty arrays
for `pending_open_channels`, `pending_closing_channels`,
`pending_force_closing_channels`, and `waiting_close_channels`.

If a channel is already opening or closing, **stop and perform the rotation later.** This
makes the before/after verification unambiguous.

---

## Step 5 — Record your exact channel inventory

Do not just record the *number* of channels. Record the actual channel points.

```bash
cd ~/btcpayserver-docker

./bitcoin-lncli.sh listchannels \
  | grep '"channel_point"' \
  | sed 's/.*: "//; s/".*//' \
  | sort \
  > "$WORK/channels-before.txt"

echo "Open channels:"
wc -l "$WORK/channels-before.txt"

cat "$WORK/channels-before.txt"
```

Keep this file. You will compare it with the post-rotation inventory.

---

## Step 6 — Copy your Static Channel Backup off the server

If you currently have Lightning channels, `channel.backup` should exist:

```bash
docker exec "$LND" ls -lh "$M/channel.backup"
```

Copy it out of the container:

```bash
docker cp "$LND:$M/channel.backup" "$WORK/channel.backup"
ls -lh "$WORK/channel.backup"
```

Now copy it somewhere your normal SSH account can access. If your username is `ubuntu`:

```bash
cp "$WORK/channel.backup" /home/ubuntu/channel.backup
chown ubuntu:ubuntu /home/ubuntu/channel.backup
ls -lh /home/ubuntu/channel.backup
```

If your login username is not `ubuntu`, substitute your actual username and home directory.

On your own computer, open a **second Terminal window** and run:

```bash
scp ubuntu@YOUR_SERVER_IP:/home/ubuntu/channel.backup \
  ~/channel.backup-$(date +%F-%H%M%S)
```

*(On older Windows using PuTTY, use PSCP or WinSCP instead.)*

> The timestamped filename prevents overwriting an SCB you saved previously.
>
> **Keep the previous copy until you have verified the new one.** Then clearly identify the
> newest SCB as your current backup — `channel.backup` changes as channels open and close,
> and **older SCBs may not include newer channels.** Your off-node copy should be kept
> current.

Verify the file exists locally and the size matches the server copy:

```bash
ls -lh ~/channel.backup-*
```

Return to your server terminal and remove the temporary copy:

```bash
rm /home/ubuntu/channel.backup
```

Do not delete the copy inside the LND data directory.

**If you have open Lightning channels but `channel.backup` is missing, stop.**

---

## Step 7 — Fingerprint the existing macaroons

The regenerated macaroon files may have the **same file sizes** as the originals. SHA-256
fingerprints let you prove the credentials actually changed without exposing them.

```bash
docker exec "$LND" sha256sum \
  /data/admin.macaroon \
  /data/invoice.macaroon \
  /data/readonly.macaroon \
  "$M/chainnotifier.macaroon" \
  "$M/invoices.macaroon" \
  "$M/router.macaroon" \
  "$M/signer.macaroon" \
  "$M/walletkit.macaroon" \
  > "$WORK/hashes-before.txt"

cat "$WORK/hashes-before.txt"
```

These hashes are fingerprints, not bearer credentials.

**Do not post the actual macaroon files, macaroon hex, connection strings, seed words, or
Lightning QR codes anywhere.**

---

## Step 8 — Delete the old macaroon files and root-key database

You have now reached the destructive part of the procedure.

This block deletes only the known macaroon credential files and `macaroons.db`. It does not
delete `wallet.db`, `channel.backup`, `walletunlock.json`, `channel.db`, or the TLS
certificate.

```bash
# Three primary LND macaroons
docker exec "$LND" rm -f /data/admin.macaroon
docker exec "$LND" rm -f /data/invoice.macaroon
docker exec "$LND" rm -f /data/readonly.macaroon

# Restricted LND subserver macaroons
docker exec "$LND" rm -f "$M/chainnotifier.macaroon"
docker exec "$LND" rm -f "$M/invoices.macaroon"
docker exec "$LND" rm -f "$M/router.macaroon"
docker exec "$LND" rm -f "$M/signer.macaroon"
docker exec "$LND" rm -f "$M/walletkit.macaroon"

# Macaroon root-key database
docker exec "$LND" rm -f "$M/macaroons.db"
```

`rm -f` normally prints nothing when successful.

**Do not restart yet.**

---

## Step 9 — Mandatory checkpoint before restart

```bash
echo "=== /data ==="
docker exec "$LND" ls -la /data/

echo
echo "=== mainnet directory ==="
docker exec "$LND" ls -la "$M/"
```

**Must be gone:**

```text
/data/admin.macaroon
/data/invoice.macaroon
/data/readonly.macaroon

chainnotifier.macaroon
invoices.macaroon
router.macaroon
signer.macaroon
walletkit.macaroon

macaroons.db
```

**Must still exist:**

```text
wallet.db
walletunlock.json
channel.backup
tls.cert
tls.key
lnd.conf
```

`walletunlock.json` is outside the scope of this rotation. **Do not delete or modify it.**

If any KEEP file is unexpectedly missing, **stop before restarting.**

---

## Step 10 — Restart LND

```bash
docker restart "$LND"
sleep 90
```

The delay gives LND time to start, unlock its wallet, recreate its macaroon database, and
generate the macaroon files.

---

## Step 11 — Verify that LND regenerated everything

```bash
docker exec "$LND" ls -la /data/
docker exec "$LND" ls -la "$M/"
```

You should again see all eight macaroon files plus `macaroons.db`, with fresh timestamps
from the current restart.

Verify that LND's admin macaroon works:

```bash
cd ~/btcpayserver-docker
./bitcoin-lncli.sh getinfo
```

Confirm `"synced_to_chain": true`.

The fact that `getinfo` succeeds proves `lncli` can authenticate using the newly generated
credentials.

---

## Step 12 — Prove that the macaroons changed

```bash
docker exec "$LND" sha256sum \
  /data/admin.macaroon \
  /data/invoice.macaroon \
  /data/readonly.macaroon \
  "$M/chainnotifier.macaroon" \
  "$M/invoices.macaroon" \
  "$M/router.macaroon" \
  "$M/signer.macaroon" \
  "$M/walletkit.macaroon" \
  > "$WORK/hashes-after.txt"

diff -u "$WORK/hashes-before.txt" "$WORK/hashes-after.txt"
```

**You want output from this diff.** Every macaroon hash should have changed.

In `diff -u` output, lines beginning with `-` are the previous hashes and lines beginning
with `+` are the new ones.

If the hashes are identical and `diff` prints nothing, **stop.** The expected credential
rotation did not occur.

---

## Step 13 — Verify the Lightning channels

```bash
cd ~/btcpayserver-docker

./bitcoin-lncli.sh listchannels \
  | grep '"channel_point"' \
  | sed 's/.*: "//; s/".*//' \
  | sort \
  > "$WORK/channels-after.txt"

diff -u "$WORK/channels-before.txt" "$WORK/channels-after.txt"
```

### Ideal result

**No output.** The same open-channel inventory exists before and after the rotation.

Then confirm:

```bash
./bitcoin-lncli.sh pendingchannels
```

If everything remains empty, that is a clean pass.

### If the channel diff is not empty

**Do not restore anything.**

A Lightning peer can independently close a channel during the maintenance window. Find out
what happened:

```bash
./bitcoin-lncli.sh pendingchannels
```

A missing open channel may now appear under `waiting_close_channels` or
`pending_force_closing_channels`. If the close already confirmed, it may instead be in
LND's closed-channel history.

**Reconciliation check** — if the close is still pending, a channel point missing from
`listchannels` should appear in `pendingchannels`. If it does not, check `closedchannels`
before treating it as unexplained.

```bash
REMOVED=$(comm -23 "$WORK/channels-before.txt" "$WORK/channels-after.txt")
CLOSING=$(./bitcoin-lncli.sh pendingchannels \
  | grep '"channel_point"' | sed 's/.*: "//; s/".*//')

echo "Missing from listchannels:"; echo "$REMOVED"
echo "Present in pendingchannels:"; echo "$CLOSING"
```

A cooperative close can confirm quickly and move straight into closed-channel history, so
also check there:

```bash
./bitcoin-lncli.sh closedchannels \
  | grep '"channel_point"' \
  | sed 's/.*: "//; s/".*//' \
  | sort
```

If the missing channel point appears in either list, it is accounted for. If it appears in
**neither**, investigate before doing anything else.

### Reading the close correctly

⚠️ **The `initiator` field does not tell you who closed the channel.** It describes who
originally *opened* it. `INITIATOR_REMOTE` simply means the peer funded the channel.

To determine what kind of close occurred, read **`chan_status_flags`**:

| Flag | Meaning |
|---|---|
| `ChanStatusCoopBroadcasted` | Cooperative close — both sides agreed. Normal, funds settle at current balances. |
| `ChanStatusRemoteCloseInitiator` | The **remote peer** initiated the close |
| `ChanStatusLocalCloseInitiator` | **Your node** initiated the close |
| `ChanStatusCommitBroadcasted` | A commitment transaction was broadcast — associated with force closes |
| `ChanStatusBorked` / `ChanStatusLocalDataLoss` | Serious. Seek help before acting. |

Multiple flags appear together separated by `|`. For example:

```text
"chan_status_flags": "ChanStatusCoopBroadcasted|ChanStatusRemoteCloseInitiator"
```

means a cooperative close initiated by the remote peer. This is a normal close type, though
an unexpected close is still worth investigating.

The macaroon commands in this guide do not issue a channel-close RPC, but any unexpected
channel-state change should still be investigated before declaring the maintenance
complete.

**Never respond to an unexplained channel close by restoring an old `channel.db`, old data
directory, or old VPS snapshot.**

---

## Step 14 — Reconnect applications

Replacing the macaroon root key invalidates all previously issued macaroons.

### BTCPay itself

BTCPay uses the macaroon from its mounted LND data path and should normally reconnect
automatically.

Check **Store → Settings → Lightning** and confirm the connection works.

### RTL

**If you do not use RTL, skip this subsection.**

```bash
docker restart "$RTL"
```

If `$RTL` is empty or unset, do not run that command — find the name first:

```bash
docker ps --format '{{.Names}}' | grep -i rtl
```

Wait briefly, then reload RTL.

RTL is a useful control: it reads the same `admin.macaroon` from the same volume. If RTL
reconnects successfully, it confirms that an internal consumer can authenticate using the
regenerated admin macaroon. If another app still fails, troubleshoot that app's connection
path separately.

(The authoritative proof that the rotation itself succeeded is the hash comparison in
Step 12, not any single application connecting.)

### Zeus, Zap, and other remote wallets

Generate a fresh connection from **Server Settings → Services → LND-gRPC** or
**LND-REST**, then scan the new QR code.

If the new connection does not work:

1. Generate a fresh LND-gRPC or LND-REST connection in BTCPay. Do not reuse an old QR
   screenshot.
2. If the wallet app supports replacing the existing connection, update it. If that fails,
   delete the old node entry and add the fresh connection as a new node.
3. If RTL and `lncli getinfo` both work but the mobile wallet does not, troubleshoot the
   external connection path rather than repeating the macaroon rotation.

### Custom API integrations

Any stored macaroon created under the old root key is now invalid. Re-bake or re-export the
appropriate restricted macaroon and update the integration.

### Other BTCPay services

If another integrated service reports an authentication failure after rotation, restart
that service so it reloads the macaroon file.

---

## Step 15 — Test a real Lightning payment

This is the final test.

1. Create a small Lightning invoice in one of your BTCPay stores (~100 sats)
2. Pay the invoice from an external Lightning wallet
3. Confirm the payment settles
4. Confirm BTCPay marks the invoice as paid

This validates the complete path:

```text
BTCPay -> LND authentication -> invoice creation -> Lightning network
       -> settlement -> BTCPay payment notification
```

If this works, the rotation is complete.

---

## Step 16 — Refresh your off-server channel backup

LND updates `channel.backup` as channel state changes.

After maintenance is complete, copy the current SCB off the server again using the same
procedure as Step 6. Keep the current copy securely with your recovery material.

---

## Troubleshooting

### `getinfo` says `root key with id 0 doesn't exist`

The client is probably still presenting a macaroon signed with the old root key.

Confirm the new files exist:

```bash
docker exec "$LND" ls -la /data/
docker exec "$LND" ls -la "$M/"
```

Then restart or reconnect the affected application so it loads the new credential. **Do not
restore old LND data.**

### The macaroon files did not return

Wait another two minutes, then inspect the startup logs:

```bash
docker logs --tail 100 "$LND"
```

LND logs many repeated `BTWL: Received non-standard input` warnings that can hide the
relevant lines. Filter them:

```bash
docker logs "$LND" 2>&1 \
  | grep -vi 'BTWL: Received non-standard input' \
  | tail -80
```

A `BTWL: Received non-standard input` warning is not, by itself, evidence of a macaroon
authentication failure.

Look specifically for wallet-unlock, macaroon, database, or startup errors.

**Do not delete or modify `walletunlock.json`.**

### The macaroon files returned but the hashes did not change

Stop. Verify that `/data/data/chain/bitcoin/mainnet/macaroons.db` was actually removed
before LND restarted.

Do not repeatedly delete files until you understand what happened.

### The peer count is lower after restart

Normal immediately after restarting LND. Peers reconnect asynchronously over several
minutes.

Use channel inventory and `pendingchannels` to determine channel state rather than relying
on peer count.

### One of my channels disappeared from `listchannels`

```bash
./bitcoin-lncli.sh pendingchannels
```

- Under `waiting_close_channels` — a closing transaction is in progress
- Under `pending_force_closing_channels` — unilateral-close resolution is underway
- Nowhere — check LND's closed-channel history and logs before taking any action

Read `chan_status_flags` (see Step 13) to determine the type of close and who initiated it.
Do not use the `initiator` field for this; it describes who opened the channel.

**Never respond to an unexplained channel-state change by restoring an old `channel.db` or
VPS snapshot.**

### A wallet app still will not connect, but RTL works

RTL connects over the internal Docker network; mobile wallets connect across the internet.
These are different paths.

**If `lncli getinfo` and RTL both work, the regenerated admin macaroon is working locally.**
The rotation succeeded. Generate a fresh connection from BTCPay and troubleshoot the
external endpoint, TLS, reverse proxy, or wallet configuration.

**Do not repeat the macaroon rotation.** It will not fix an external routing or endpoint
problem, and each rotation invalidates every credential again.

Useful checks that require no credential handling:

- Load your BTCPay domain in the device's browser over cellular data, WiFi off — confirms
  the server is reachable from outside your network
- Compare the URL the wallet is using against exactly what BTCPay's service page displays.
  Do not reconstruct or guess the path.
- Try LND-gRPC instead of LND-REST, or vice versa

> Avoid copying macaroons out of the container for ad-hoc testing. Writing them to
> temporary files or passing them on a command line exposes a full admin credential through
> file permissions, shell history, and the process list.

---

## Security reminders

Treat the following as secrets:

- LND seed
- macaroon files
- macaroon hex
- BTCPay LND connection strings
- LND QR codes

Do not post them in chat, support tickets, forums, GitHub issues, email, logs, or
screenshots.

**A SHA-256 hash of a macaroon is safe for before/after comparison. The macaroon itself is
not.**

---

## What this guide deliberately does not rotate

- LND TLS certificate
- LND TLS private key
- Bitcoin wallet seed
- wallet database
- channel database
- Lightning channels

TLS rotation should be treated as a separate maintenance procedure so authentication
problems are not mixed with certificate problems.

---

## Reference and tested configuration

This procedure follows the same basic credential-reset model documented in the
[BTCPay Server Lightning FAQ](https://docs.btcpayserver.org/FAQ/LightningNetwork/), but
adds:

- configuration verification before deletion
- off-server Static Channel Backup
- exact channel-point capture and reconciliation
- macaroon fingerprint verification
- the five restricted LND subserver macaroon files observed on the tested LND v0.21 BTCPay
  deployment
- explicit warning about custom-macaroon invalidation
- correct interpretation of `chan_status_flags` versus `initiator`
- post-rotation application reconnection
- end-to-end payment verification

Tested on a LunaNode BTCPay Server Docker deployment running Bitcoin mainnet and LND
`v0.21.0-beta`.

**If your version, database backend, macaroon paths, or filesystem layout differ from the
checks in this guide, stop rather than adapting the deletion commands blindly.**
