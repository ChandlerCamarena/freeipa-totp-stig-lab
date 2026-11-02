# FreeIPA with TOTP MFA on RHEL 9, enrolled into a DISA STIG hardened client

Hands-on lab: a FreeIPA identity server on RHEL 9.8 (FIPS mode) acting as the central identity provider, with TOTP two-factor authentication enforced for a test user, and a DISA STIG hardened RHEL 9.8 host enrolled as a client. MFA login was demonstrated on the hardened client (success and failure), and the effect of client enrollment on the STIG baseline was measured with OpenSCAP and reconciled.

Everything in this README was run in the lab. Where the evidence cannot prove something, the text says so.

## Results at a glance

- FreeIPA 4.13.4 server with integrated CA (Dogtag) and integrated DNS (BIND), FIPS mode enabled at install time.
- Test user `mfauser` restricted to the `otp` authentication type with a self-enrolled TOTP token.
- Server side proof: password plus TOTP accepted (`pa_type 141`, OTP preauthentication), same-step retry rejected, wrong second factor rejected.
- STIG hardened client `rhel9-stig` enrolled with `ipa-client-install`, using a one-time host enrollment password and a CA certificate supplied out of band.
- MFA SSH login to the hardened client: `First Factor` and `Second Factor` prompts, success and failure logged on both the client and the KDC.
- Enrollment dropped the client's STIG score from 97.04 (9 failing rules) to 93.00 (14 failing). After reconciliation: 94.10 (11 failing), consisting of the 9 pre-existing failures plus 2 documented package deviations.

## Architecture

```
                    libvirt NAT 192.168.122.0/24
                    gateway / dnsmasq 192.168.122.1
                               |
         +---------------------+----------------------+
         |                                            |
 ipa.lab.test  192.168.122.10               rhel9-stig.lab.test  192.168.122.229
 RHEL 9.8, FIPS                             RHEL 9.8, FIPS:STIG crypto policy
 389 DS (LDAP)                              DISA STIG hardened (OpenSCAP + Ansible)
 MIT Kerberos KDC + ipa-otpd                fapolicyd enforcing, /home noexec
 Dogtag CA, httpd (web UI, API)             SSSD (IPA provider), authselect
 BIND: authoritative for lab.test,          resolver: 192.168.122.10
   forwards everything else to .1
```

Authentication path for an OTP user logging in over SSH:

```
ssh client -> sshd (keyboard-interactive) -> PAM pam_sss -> SSSD
  -> Kerberos AS-REQ, FAST armored -> KDC on ipa.lab.test
  -> KDC hands the OTP request to ipa-otpd over a local Unix socket (RADIUS format)
  -> ipa-otpd does an LDAP bind as the user with password + code
  -> 389 DS ipa-pwd-extop validates the password and the TOTP code
  -> Access-Accept or Access-Reject back to the KDC -> ticket issued or PREAUTH_FAILED
```

## Environment

| Item | Value |
|---|---|
| Host | Arch Linux, KVM/QEMU, libvirt, virt-manager |
| Network | libvirt default NAT, DHCP reservations for all VMs |
| IPA server VM | RHEL 9.8 (kernel 5.14.0-687.64.1.el9_8), 4 GiB RAM, 2 vCPU, 40 GB, UEFI, installer booted with `fips=1`, Minimal Install |
| Client VM | RHEL 9.8, DISA STIG hardened in a previous lab (OpenSCAP STIG profile, final score 97.04 before this lab), crypto policy `FIPS:STIG` |
| FreeIPA | `ipa-server` / `ipa-client` 4.13.4-1.el9_8 (AppStream, no module stream) |
| SSSD | 2.9.8-4.el9_8.1 |
| Kerberos | krb5 1.21.1-10.el9_8.1 |
| Subscription | Red Hat Developer Subscription |
| IPA domain / realm | `lab.test` / `LAB.TEST` |

Domain choice: a subdomain of a real domain was considered first, but that domain has a public wildcard DNS record, so any IPA name would also resolve publicly to the wrong address. `.test` is reserved by RFC 6761 and never resolves publicly.

## Server installation

Packages were verified from the enabled repositories before installing (`dnf info ipa-server ipa-server-dns`).

```bash
sudo hostnamectl set-hostname ipa.lab.test
echo "192.168.122.10 ipa.lab.test ipa" | sudo tee -a /etc/hosts
sudo dnf -y install ipa-server ipa-server-dns
sudo firewall-cmd --permanent --add-service={freeipa-4,dns} && sudo firewall-cmd --reload
sudo ipa-server-install \
  --realm LAB.TEST --domain lab.test --hostname ipa.lab.test \
  --ip-address 192.168.122.10 \
  --setup-dns --forwarder 192.168.122.1 --auto-reverse --no-dnssec-validation
```

Decisions:

- **FIPS at OS install time.** Switching an installed IdM server into FIPS mode is not supported, and the client already runs in FIPS mode. Observed result: tickets use `aes256-cts-hmac-sha384-192`.
- **`--ip-address`** pins the address. `getent hosts ipa.lab.test` returned the IPv6 link-local address via the `myhostname` NSS module.
- **Integrated DNS** makes IPA authoritative for `lab.test` and maintains its SRV records. Non-IPA names are forwarded to libvirt's dnsmasq (`Forward policy: only`).
- **`--no-dnssec-validation`**: the upstream forwarder (dnsmasq) does not validate DNSSEC.
- **Passwords entered at prompts**, not on the command line.
- **Reverse zone:** `--auto-reverse` created none, because dnsmasq already answered the PTR for its own DHCP reservation. The zone was created afterward:

```bash
ipa dnszone-add 122.168.192.in-addr.arpa.
ipa dnsrecord-add 122.168.192.in-addr.arpa. 10 --ptr-rec ipa.lab.test.
```

## Server verification

- `ipactl status`: all nine services RUNNING, including `ipa-otpd`, before and after a cold reboot.
- `kinit admin`, `klist -e`: `aes256-cts-hmac-sha384-192` for both the session key and the ticket.
- `ipa server-role-find`: CA server and DNS server roles enabled.
- DNS: `_kerberos._udp` and `_ldap._tcp` SRV records, A records for `ipa` and `ipa-ca`, forwarding (an external name resolved), and the PTR for 192.168.122.10.
- Web UI reachable from the host, validated against the IPA CA (SHA-256 fingerprint `76:06:2C:C0:54:65:BE:47:59:08:DD:9B:21:1D:AC:7C:90:74:AE:43:F4:9E:50:D3:1B:07:E5:6D:9D:55:FA:88`, compared on both sides). The CA was imported into the browser only, not the host's system trust store.

## User and TOTP policy

How FreeIPA OTP works:

- **The `otp` user authentication type** means password plus OTP code.
- **No-token bootstrap.** If `otp` is the only type and the user has no token, the KDC still accepts the password alone, so the user can log in once and self-enroll. The KDC logged this explicitly: `does not have a valid token and only OTP authentication is supported, checking for expired tokens...`, then issued a ticket.
- **FAST armor.** OTP preauthentication runs inside a FAST tunnel. Here the armor ticket came from anonymous PKINIT (`kinit -n`), which worked in FIPS mode.
- **Where the code is checked.** The KDC passes the OTP request to `ipa-otpd`, which does an LDAP bind as the user with the password and code joined into one string.

```bash
ipa user-add mfauser --first=MFA --last=Demo --shell=/bin/bash --password
kinit mfauser                    # admin-set password is pre-expired; user sets their own
ipa user-mod mfauser --user-auth-type=otp
kinit -n @LAB.TEST -c FILE:/tmp/armor.ccache
kinit -T FILE:/tmp/armor.ccache -c FILE:/tmp/mfa.ccache mfauser
```

- **Per-user OTP.** Enforced for `mfauser` only, not globally, so `admin` is not locked out on a single-server lab.
- **Self-enrollment.** The token was enrolled by the user in the web UI (Authentication, OTP Tokens, Add, TOTP, SHA-1, 6 digits, 30 seconds).

### Server side OTP proof

At the `Enter OTP Token Value:` prompt, the value entered is the password immediately followed by the 6-digit code.

| Time | Attempt | ipa-otpd | KDC / ticket |
|---|---|---|---|
| 21:34:42 | password only, no token yet (bootstrap) | not involved | ticket, `pa_type 138` (encrypted challenge) |
| 21:46:33 | password + current code | `bind end: success`, `Access-Accept` | ticket, `pa_type 141` (OTP) |
| 21:46:46 | second attempt in the same 30 second TOTP step, intended as a replay of the same code | `Invalid credentials`, `Access-Reject` | `Preauthentication failed` |
| 21:47:01 | wrong second factor | `Access-Reject` | `Preauthentication failed` |

The logs record the rejections but cannot show which digits were typed. The replay row is labeled by intent for that reason.

## Client enrollment

### Pre-flight on the hardened client

- **Read-only checks:** authselect profile, faillock settings, sshd drop-in order, the SSSD config, `/home` mount options, and TCP reachability of ports 53, 88, 389, 443, 464 and 636.
- **Backup:** `/etc/pam.d`, `/etc/authselect`, `/etc/nsswitch.conf`, `/etc/ssh`, `/etc/security`, `/etc/sssd` and the Kerberos config were archived for later comparison.
- **Snapshots:** a VM snapshot was taken before enrollment, and a root shell was kept open in a second session while PAM changed.

### DNS and hostname

```bash
sudo nmcli con mod enp1s0 ipv4.dns 192.168.122.10 ipv4.ignore-auto-dns yes ipv4.dns-search lab.test
sudo nmcli device reapply enp1s0
sudo hostnamectl set-hostname rhel9-stig.lab.test
```

### Host entry and enrollment

On the IPA server, the host entry is created with A and PTR records and a one-time enrollment password, so admin credentials are never typed on the client:

```bash
ipa host-add rhel9-stig.lab.test --ip-address=192.168.122.229 --random
```

On the client, the CA certificate was copied from the admin host and its fingerprint verified before use:

```bash
sudo dnf install -y ipa-client oddjob-mkhomedir
read -rs -p "Enrollment OTP: " OTP; echo
sudo ipa-client-install --domain lab.test --mkhomedir --no-ntp \
  --ca-cert-file /root/lab-test-ca.crt --password "$OTP" --unattended
unset OTP
```

- **`--domain` without `--server`** exercises DNS SRV discovery (`Discovery was successful!`).
- **`--no-ntp`** leaves the STIG chrony configuration untouched.
- **The first enrollment attempt failed** because a placeholder string was passed as the one-time password. That attempt fetched the CA over plain HTTP (`Downloading the CA certificate via HTTP, this is INSECURE`) because no CA file was supplied. Its rollback moved the existing STIG `sssd.conf` to `sssd.conf.deleted` instead of restoring the prior state. The client was reverted to the pre-enrollment snapshot and enrolled again with `--ca-cert-file`; the HTTP warning did not appear.

## Interaction with the STIG hardening

### What enrollment changed, and what was done

| Area | Effect of enrollment | Action |
|---|---|---|
| authselect | Profile `custom/hardening` (with-faillock, without-nullok, with-smartcard, with-fingerprint, with-silent-lastlog) was replaced by `sssd` with `with-mkhomedir` and `with-sudo`. Account lockout, the nullok restriction and smartcard auth were gone. | Reselected `custom/hardening` with the five original features plus `with-mkhomedir` and `with-sudo`. `authselect check` reports a valid configuration. A diff of `/etc/authselect` against the pre-enrollment backup shows only the mkhomedir session line, `sudoers: files sss`, and the feature list. |
| sshd | IPA wrote `04-ipa.conf`. The STIG file `01-complianceascode-reinforce-os-defaults.conf` sorts first, so `GSSAPIAuthentication no` stayed in effect (`sshd -T`). IPA's `ChallengeResponseAuthentication yes` took effect over Red Hat's `50-redhat.conf`, enabling keyboard-interactive. | Changed `GSSAPIAuthentication` in `04-ipa.conf` to `no` so the file matches the effective setting (the scanner reads file text). |
| SSSD | New `sssd.conf` with cached credentials and no expiry. `pam_cert_auth = True` was carried over by the installer. | Added `/etc/sssd/conf.d/60-stig-offline-cred.conf` with `offline_credentials_expiration = 1`. |
| Services | `autofs` installed; `rpcbind` enabled and listening on port 111. | `autofs` masked (the rule requires masked, not just disabled). `rpcbind` disabled; port 111 closed. The rpcbind change is beyond what the scan flagged. |
| setuid auditing | A sweep of setuid/setgid binaries on the root filesystem against the loaded audit rules found 15 without execution auditing. Package install dates show 2 came from the IPA client's dependencies (`ksu` from krb5-workstation, `mount.nfs` from nfs-utils) and 13 predate this lab. | Added `/etc/audit/rules.d/60-setuid-exec-audit.rules` for all 15. Audit rules are immutable under the STIG (`enabled 2`), so they were loaded by reboot. The OpenSCAP STIG profile did not flag this gap, so it is defense in depth, not a scanned finding. |
| fapolicyd | No `FANOTIFY` audit records after enrollment. The rules use `deny_audit`, so denials would have been recorded. | None needed. |
| `/home` noexec | Home directory creation and IPA user login worked. | None needed. |
| faillock | Applies to IPA users on the client (a failed `mfauser` login appeared in `faillock --user mfauser`). | Reset after the demo. |

### OpenSCAP STIG profile scores on the client

Same profile (`xccdf_org.ssgproject.content_profile_stig`) and data stream (`ssg-rhel9-ds.xml`) as the original hardening lab.

| Scan | Score | Failing rules |
|---|---|---|
| Final STIG scan before IPA | 97.04 | 9 |
| Immediately after enrollment | 93.00 | 14 |
| After reconciliation | 94.10 | 11 |

New failures right after enrollment: `sshd_disable_gssapi_auth`, `service_autofs_disabled`, `sssd_offline_cred_expiration`, `package_nfs-utils_removed`, `package_gssproxy_removed`.

Remaining deviations, justified:

- **`package_nfs-utils_removed` and `package_gssproxy_removed`:** `ipa-client` requires `nfs-utils`, which requires `gssproxy`. `dnf remove --assumeno gssproxy nfs-utils` shows `ipa-client` removed as a dependent package. The gssproxy service is disabled, autofs is masked, and rpcbind is disabled.

The 9 pre-existing failures are documented in the STIG hardening lab and are unchanged by this lab.

## MFA login on the hardened client

From the admin host:

```
$ ssh mfauser@192.168.122.229
(mfauser@192.168.122.229) First Factor:
(mfauser@192.168.122.229) Second Factor:
[mfauser@rhel9-stig ~]$ id; pwd; klist
uid=1312600003(mfauser) gid=1312600003(mfauser) groups=1312600003(mfauser) context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
/home/mfauser
Default principal: mfauser@LAB.TEST
11/01/2026 23:27:35  11/02/2026 22:48:10  krbtgt/LAB.TEST@LAB.TEST
```

A second login with an incorrect second factor was rejected.

| Time | Client `/var/log/secure` | Server `ipa-otpd` | Server `krb5kdc.log` (from 192.168.122.229) |
|---|---|---|---|
| 23:27:35 | `pam_sss(sshd:auth): authentication success ... user=mfauser`, `Accepted keyboard-interactive/pam for mfauser` | `bind end: success`, `Access-Accept` | `ISSUE ... mfauser@LAB.TEST for krbtgt/LAB.TEST`, then `ISSUE ... for host/rhel9-stig.lab.test` |
| 23:28:07 | `pam_sss(sshd:auth): authentication failure ... user=mfauser`, `Failed keyboard-interactive/pam for mfauser` | `bind end: Invalid credentials`, `Access-Reject` | `PREAUTH_FAILED: mfauser@LAB.TEST ... Preauthentication failed` |

**What the evidence shows:**
- **Authentication was decided centrally.** The client only relayed it: the KDC log carries the client's IP address, and `ipa-otpd` made the accept or reject decision.
- **SSSD validated the ticket against the client's own keytab.** The second KDC `ISSUE` line, for `host/rhel9-stig.lab.test`, is that validation.
- **The client's faillock recorded the failed login** (`faillock --user mfauser`).

## Findings

1. **A routine `ipa-client-install` silently removes STIG controls** (faillock, nullok restriction, smartcard) on a host using a custom authselect profile. Compliance has to be rechecked after identity enrollment, not assumed.
2. **Enrolling with a one-time password and no `--ca-cert-file`** downloads the IPA CA over unauthenticated HTTP. Supplying a CA whose fingerprint was verified out of band removes that trust-on-first-use step.
3. **A failed client install's rollback leaves residue.** Here it moved the original `sssd.conf` aside rather than restoring it. A VM snapshot was the real rollback.
4. **The STIG scanner checks configuration text, not effective daemon configuration.** `GSSAPIAuthentication yes` in a later-sorted drop-in failed the rule, while `sshd -T` showed `no`.
5. **An IPA admin can read every user's TOTP seed** (`ipa otptoken-find --all` printed the key). Seeds are symmetric secrets stored in LDAP, so compromise of `admin` or an LDAP backup allows minting codes for any user.
6. **LDAP bind failures count toward the IPA account lockout**, not only Kerberos failures.
7. **`/var/log/httpd/error_log` on the IPA server** records every API call with the calling principal and result. It is a useful audit source.

## Limitations

- **Infrastructure scope:** single IPA server, no replica. The IPA server itself was not STIG hardened. NAT lab network; `lab.test` is not delegated from a public parent.
- **DNSSEC validation** is disabled on the IPA DNS server because the upstream forwarder does not validate.
- **HBAC:** the default `allow_all` HBAC rule was left enabled; host access was not restricted per user or group.
- **SELinux:** IPA users map to `unconfined_u` on the client; no SELinux user map was configured.
- **OTP model:**
  - The no-token bootstrap window lets a user with `otp` auth type and no token log in with a password alone.
  - TOTP uses symmetric seeds shared between IdM and the token. That conflicts with a strict FIPS interpretation even though the system runs in FIPS mode.
- **Lab-only password handling:** `mfauser`'s password was reset by an admin and its expiry extended, rather than forcing a user change at next login.
- **Unusable CA backup:** `/root/cacert.p12` is encrypted with the original Directory Manager password, which was lost and reset during the lab, so that file is not usable.
- **Out of scope:** forwarding these logs to Splunk is not part of this repository.

## Operational notes

- **Lost admin password, recovered.** It was reset as root over LDAPI using SASL EXTERNAL autobind, which maps root to Directory Manager, via `ldappasswd` (password modify extended operation). IPA's password plugin regenerated the Kerberos keys. Root on an IPA server is effectively Directory Manager.
- **Lost Directory Manager password, recovered.** It was reset online the same way by replacing `nsslapd-rootpw` in `cn=config` with `ldapmodify -Y EXTERNAL`. 389 DS stored it as `{PBKDF2-SHA512}`.
- **Password mismatch traced to input.** Kerberos rejected a manually typed password that LDAP accepted. After a reset with a password-manager value pasted at the prompts, both agreed.
- **Leaked TOTP seed revoked.** A TOTP seed was printed into a terminal session log during testing. That token was deleted (`ipa otptoken-del`) and a new token was self-enrolled. Evidence files were checked for `otpauth://`, `secret=` and `Key:` lines before publishing.

## Reproduction (condensed)

1. Create the server VM with `virt-install` (UEFI, 4 GiB, 40 GB, MAC reserved in the libvirt network for 192.168.122.10). Boot the installer with `fips=1`. Choose Minimal Install, a static IP, and hostname `ipa.lab.test`.
2. Register the system, add the `/etc/hosts` entry, install `ipa-server ipa-server-dns`, open the firewall, and run `ipa-server-install` as above. Create the reverse zone.
3. Verify with `ipactl status`, `kinit admin`, the `dig` SRV/A/PTR checks, and the web UI.
4. Create `mfauser`, complete the first-login password change, and set `--user-auth-type=otp`. Self-enroll a TOTP token in the web UI. Test with an armored `kinit` (`kinit -n`, then `kinit -T`).
5. On the client, point DNS at 192.168.122.10, set the FQDN, and snapshot. On the server, run `ipa host-add ... --random`. On the client, install `ipa-client oddjob-mkhomedir` and run `ipa-client-install` with `--ca-cert-file`.
6. Reconcile the client:

```bash
sudo authselect select custom/hardening with-fingerprint with-silent-lastlog with-faillock \
  without-nullok with-smartcard with-mkhomedir with-sudo
sudo sed -i 's/^GSSAPIAuthentication yes$/GSSAPIAuthentication no/' /etc/ssh/sshd_config.d/04-ipa.conf
sudo sshd -t && sudo systemctl reload sshd
printf '[pam]\noffline_credentials_expiration = 1\n' | sudo tee /etc/sssd/conf.d/60-stig-offline-cred.conf
sudo chmod 600 /etc/sssd/conf.d/60-stig-offline-cred.conf && sudo systemctl restart sssd
sudo systemctl mask --now autofs.service
sudo systemctl disable --now rpcbind.socket rpcbind.service
# setuid execution audit rules for any unaudited setuid/setgid binary, then reboot (audit rules are immutable)
```

7. Rescan with `oscap xccdf eval --profile xccdf_org.ssgproject.content_profile_stig ... /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml` and compare failed rule IDs against the pre-IPA results.
8. `ssh mfauser@<client>` and answer `First Factor` (password) and `Second Factor` (TOTP code).

## Evidence

| File | Contents |
|---|---|
| `evidence/ipa-server-install-console.txt` | Server install console output and server verification |
| `evidence/webui-admin-dashboard.png` | Web UI logged in as admin |
| `evidence/webui-mfauser-token-list.png` | User token list (first token, later revoked; no QR code or secret shown) |
| `evidence/step4b*.txt` | Server side OTP tests: bootstrap, accept, same-step retry, rejects |
| `evidence/post-ipa-compare.txt` | STIG comparison right after enrollment |
| `evidence/post-ipa-fixed-compare.txt` | STIG comparison after the first fixes |
| `evidence/post-ipa-final-compare.txt` | Final STIG comparison |
| `evidence/step6-client-secure.txt` | Client `/var/log/secure` for the MFA SSH logins |
| `evidence/step6-client-faillock.txt` | Client faillock entry for the failed login |
| `evidence/step6-server-kdc.txt` | KDC log lines for logins from the client |
| `evidence/step6-server-otpd.txt` | ipa-otpd accept and reject decisions |
