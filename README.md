# ansible-hardening-freebsd

![Ansible](https://img.shields.io/badge/Ansible-2.15%2B-EE0000?logo=ansible&logoColor=white)
![FreeBSD](https://img.shields.io/badge/FreeBSD-14%20%2F%2015-AB2B28?logo=freebsd&logoColor=white)
![Hardening](https://img.shields.io/badge/security-hardening-4B275F)
![2FA](https://img.shields.io/badge/2FA-TOTP%20%2B%20FIDO2-005571)
![Tested with Vagrant](https://img.shields.io/badge/tested%20with-Vagrant-1868F2?logo=vagrant&logoColor=white)

[![License: MIT](https://img.shields.io/github/license/AcidDemon/ansible-hardening-freebsd?color=blue)](LICENSE)
[![Last commit](https://img.shields.io/github/last-commit/AcidDemon/ansible-hardening-freebsd)](https://github.com/AcidDemon/ansible-hardening-freebsd/commits/main)
[![Repo size](https://img.shields.io/github/repo-size/AcidDemon/ansible-hardening-freebsd)](https://github.com/AcidDemon/ansible-hardening-freebsd)
[![GitLab mirror](https://img.shields.io/badge/mirror-GitLab-FC6D26?logo=gitlab&logoColor=white)](https://git.inviziblenet.work/AcidDemon/ansible-hardening-freebsd)

Hardens a freshly installed FreeBSD host in one idempotent run: accounts, SSH and 2FA,
sshguard and CrowdSec on pf, OpenBSM audit, AIDE, Lynis, chkrootkit, and a daily posture
report. Runs after `ansible-base-freebsd`, which owns pf, users, and time.

## Roles

| Role | What it does |
|---|---|
| `accounts` | login.conf policy (umask, passwd_format, expiry), sudo hardening drop-in |
| `ssh_hardening` | `sshd_config.d` drop-in, AllowGroups self-heal, tight crypto, ECDSA excluded by algorithm |
| `crowdsec` | agent + pf firewall bouncer, whitelists `management_cidrs`, installs the sshd/linux collections |
| `sshguard` | log-driven SSH banning through base's pf `<sshguard>` table |
| `sysctl_hardening` | posture sysctls layered over base's baseline |
| `auditd` | OpenBSM audit classes |
| `aide` | file-integrity database with a daily periodic check that says when the baseline itself is broken |
| `lynis` | weekly audit from periodic (`460.lynis`): every warning, suggestions as a count |
| `hardening_misc` | coredumps off, cron/at allowlists, optional noexec `/tmp` and mail relay |
| `rootkit_scanners` | chkrootkit, daily from periodic security (`530.chkrootkit`) |
| `security_reporting` | daily posture report (sockstat, pfctl, cscli, pkg audit), every section labelled when empty |
| `ssh_2fa` | TOTP and FIDO2 tiers; the deploy account stays key-only |

## Run
    ansible-galaxy collection install -r requirements.yml
    ansible-playbook -i inventory/hosts.yml site.yml

Chain order is base, then app, then hardening. In a service repo, end `site.yml` with
`import_playbook: acidnetworks.hardening_freebsd.harden`. Set `management_cidrs` in
`inventory/group_vars/all/main.yml` to your control host's network, or the SSH cutover
locks you out.

## Lockout safety

`harden_safe_rollback` (default true) schedules an `at(1)` job before each SSH cutover,
ssh_hardening and then ssh_2fa. If the run loses the session, that job strips the SSH
drop-ins, puts back the stock `/etc/pam.d/sshd` and reloads sshd after
`harden_rollback_minutes` (default 20, plus up to 5 for atrun). It gets cancelled once
connectivity is re-checked, and the play fails if it had already fired. The roles between
the two cutovers touch neither sshd nor PAM and run unarmed, so a window only has to outlast
its cutover, not a first run's `aide --init`.
A probe runs after ssh_hardening and after ssh_2fa, and every sshd drop-in is checked with
`sshd -t -f` before it lands. The rollback never touches pf. The probe reconnects as the
deploy account, which 2FA exempts, so it cannot see a 2FA change that locks out only people.
Keep a second session open when you change 2FA settings.

## 2FA enrollment

2FA deploys with `ssh_2fa_totp_nullok: true`, so a user without an enrolled token is not
locked out. The deploy account is always key-only and exempt. Each operator enrolls before
you set nullok to false.

TOTP, on the host as the operator:

    google-authenticator --time-based --disallow-reuse --force --rate-limit=3 --rate-time=30 --window-size=3

FIDO2, on the workstation, then add the pubkey to the host and the user to `fido2users`:

    ssh-keygen -t ed25519-sk -O verify-required -f ~/.ssh/id_ed25519_sk

The group accepts only ed25519-sk keys, and `-O verify-required` is not optional: without it
the client never asks for the PIN and sshd refuses the signature. Add a user to `fido2users`
only once that key is on the host, or they are locked out.

Once everyone is enrolled, set `ssh_2fa_totp_nullok: false` and re-run. With nullok off,
the play stops before it edits PAM if a member of `ssh_allow_group_members`, other than
`deploy_user` and `ssh_2fa_exempt_users`, has no usable `~/.google_authenticator` (missing,
owned by someone else, or a mode other than 0400 or 0600). It cannot tell whether the
codes work, so log in once from a second terminal before the run.

### Service accounts

`nullok` is what keeps unattended accounts working before that flip, and it stops
protecting them the moment you set it false. Anything that authenticates by key and
cannot answer a TOTP prompt - backup ingest, monitoring, CI - belongs in
`ssh_2fa_exempt_users` or `ssh_2fa_exempt_groups`, which render their own key-only
`Match` stanzas. `deploy_user` is always exempt. Such accounts also need to be in
`ssh_allow_group_members`, or `AllowGroups` shuts them out before 2FA is even reached.

`deploy_user` skips the code, and base gives it NOPASSWD sudo, so a copy of its key is root.
`ssh_2fa_deploy_from` pins it to the addresses or CIDRs you list, as sshd sees the source,
rendered as `AllowUsers` inside its own `Match User`. Empty, the default, accepts it from
anywhere sshd is reachable. A wrong address fails the probe after ssh_2fa, the play cannot
disarm, and the rollback strips the drop-ins when it fires.

## Jail hosts

A jail's sshd logs to the jail's own `auth.log`, which neither watcher sees by default.
Add the paths explicitly: `sshguard_log_paths` (appended to sshguard's `LOGREADER`) and
`crowdsec_acquis_paths` (written to `acquis.d/10-extra.yaml`). Bans still land in base's
shared pf tables, so a ban applies host-wide, jails included.

`ssh_client_alive_interval` / `ssh_client_alive_count_max` override the idle reaper.
The default kills a session after ~10 minutes without SSH-layer traffic, which is short
for a long `zfs recv` or a stalled backup.

## Nightly reports

base sets `security_show_success=NO`, so a periodic security check that exits 0 prints
nothing. The drop-ins here follow that contract, `530.chkrootkit` and the weekly
`460.lynis` included: `510.aide` exits 1 on a clean run and 3 when it found differences,
when there is no database, or when the database is too small to be a real baseline. That
last case matters because AIDE 0.19.0 through 0.19.2 mishandles `st_rdev` on FreeBSD: its
two stat calls on a directory disagree, so it skips recursion at `/`, writes a database
holding one entry and then reports "no differences" forever (aide/aide#208, fixed
upstream in 0.19.3). `aide_min_entries` sets the floor, and the role rebuilds any
database that falls under it.

The package is installed `state: present`, not `state: latest`. Which pkg branch a
host follows decides whether 0.19.3 is reachable at all, and that is not a role's
call: quarterly carried only 0.19.2 through 2026Q3. On a box that cannot reach the
fix, the nightly mail says so rather than the play pretending it fixed something.

The posture report labels every empty section in words instead of printing a bare
heading, and it keeps stderr in the mail. `cscli` writes "No active decisions" there,
so the section used to come out blank on exactly the days there was nothing to worry
about.

## Lockout recovery

From the console:

    rm -f /etc/ssh/sshd_config.d/10-hardening.conf /etc/ssh/sshd_config.d/20-2fa.conf
    cp /etc/pam.d/sshd.orig /etc/pam.d/sshd
    service sshd reload

Restore the PAM file too, as the rollback job does. Without `20-2fa.conf` nothing puts a key
in front of keyboard-interactive, and ssh_2fa's PAM stack then lets in a TOTP code alone, or
with nullok an account that never enrolled on an empty prompt. `sshd.orig` is the stock file
ssh_2fa saved before its first edit; if it is missing, ssh_2fa never touched PAM.

pf belongs to base. If it is blocking you, re-run `ansible-base-freebsd`. `pfctl -d`
disables pf entirely as a last resort.
