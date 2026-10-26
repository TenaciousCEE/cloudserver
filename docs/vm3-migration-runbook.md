# vm3 mail migration runbook

This is a human-guided procedure for replacing `vm3` with a clean, supported
Debian Droplet. It is deliberately procedural rather than an automated
playbook. Do not commit secrets, private keys, mailbox contents or database
dumps to this repository.

## Scope and principles

- Mail is the primary service: Postfix, Dovecot, OpenDKIM, SpamAssassin,
  ClamAV, Fail2Ban, Shorewall and Let's Encrypt.
- Build a new Droplet rather than restoring the obsolete web stack.
- Treat Apache, Joomla, MediaWiki, Piwigo and Roundcube as secondary. Roundcube
  may run on the mail host or on a separate web host with secure IMAP/SMTP
  access.
- Do not change production DNS, firewall rules or services without approval.
- Keep the old server unchanged and available until the migration is accepted.

## 1. Record the current baseline

Run read-only checks on `vm3` and save a sanitised copy outside Git:

```sh
hostnamectl
cat /etc/os-release
uname -a
df -hT
free -h
systemctl --failed
ss -lntup
```

Record the OS and kernel, Droplet size, volumes, VPC address, hostnames, mail
domains, aliases, mailbox locations, account/authentication method, and enabled
services. Also record Postfix and Dovecot settings, OpenDKIM selectors and
domains, SpamAssassin/ClamAV rules, Shorewall policies, Fail2Ban jails,
certificate names/expiry, and all DNS records (A/AAAA, MX, SPF, DKIM, DMARC).

Audit the web side separately: Apache virtual hosts, PHP versions, Joomla,
MediaWiki, Piwigo, Roundcube, databases, uploads and data locations. Treat the
legacy PHP applications as unsafe until upgraded or replaced.

## 2. Confirm DigitalOcean prerequisites

Before building the replacement:

1. Obtain written confirmation that SMTP port 25 and required submission ports
   will be available for the new Droplet.
2. Confirm region, Droplet size, VPC, volumes, backups and snapshot options.
3. Set reverse DNS for the final mail hostname and confirm matching forward
   DNS. Check the proposed IP against blocklists.
4. Decide whether the web applications will stay offline, move to another VM,
   or be rebuilt on managed hosting. Do not expose the old applications merely
   to preserve convenience.

## 3. Backups and rollback

Create and verify, before migration:

- a fresh Droplet snapshot;
- a separate copy of reviewed `/etc` mail configuration;
- mailbox data with ownership, permissions and timestamps;
- OpenDKIM private keys and corresponding public records;
- TLS/Certbot state, or a plan to issue new certificates;
- MariaDB logical dumps for Roundcube and retained web applications;
- retained website files and uploads, if required;
- package versions and enabled services.

Check sizes, checksums and archive readability. Test-restore at least one
mailbox, one database and one representative website file. A snapshot alone is
not a sufficient mailbox backup.

Snapshot retention strategy:

- Label retained snapshots unmistakably, for example `KEEP-vm3-<state>-<date>`.
- Keep a known-good snapshot during a one-to-two-week settling period after a
  major upgrade or migration.
- Once the system has remained stable, take and verify a fresh current-state
  snapshot before deleting superseded checkpoints.
- Retain one clearly labelled production recovery snapshot, while deleting
  intermediate upgrade snapshots and retired UAT snapshots when no longer
  useful.
- Never delete the old snapshot set until the replacement snapshot has
  completed successfully and its name, date and Droplet association have been
  checked.
- A Droplet snapshot is a rapid system rollback/rebuild option, not a
  substitute for independent mailbox, database and website backups. A restored
  snapshot also retains any undetected compromise in the source system.

Keep the old host powered on and unchanged. Lower DNS TTLs in advance, but do
not change MX or A records yet. If rollback is needed, restore the old DNS
records, stop new delivery on the replacement, retain its new mail for
reconciliation, and only then resume using the old host. Do not run two
independent writable copies of the same mailbox store for an extended period.

## 4. Build and isolate the new Droplet

1. Create a Droplet using a currently supported Debian release.
2. Configure hostname, VPC and any separate data volume.
3. Install the DigitalOcean agent and confirm both SSH and provider-console
   access.
4. Configure the firewall conservatively. Permit SSH through a safe recovery
   path, SMTP 25 as required, submission 587, IMAPS 993, and HTTP/HTTPS only
   for ACME or approved webmail use. Keep databases on localhost or the private
   network.
5. Configure Fail2Ban for SSH, Postfix SASL and Dovecot. Test a temporary ban
   and unban.

Keep the new host off the public MX until the full test stage. Keep test
firewall rules separate from production rules.

## 5. Install and configure mail services

Install only required packages from supported Debian repositories, then apply
the reviewed baseline incrementally:

1. Postfix: domains, aliases, relay restrictions, TLS, SASL and milters.
2. Dovecot: mailbox format/location, authentication, IMAP, TLS and userdb.
3. OpenDKIM: socket, ownership, signing table, key table and selectors.
4. SpamAssassin and ClamAV: reviewed local rules and functioning milters.
5. Certbot: certificates for final public names; keep renewal disabled until
   DNS and reachability are ready, then test renewal safely.
6. Roundcube: a supported release with IMAPS/SMTP submission and a restricted
   database account.

For package upgrades choose the conffile policy in advance. Normally retain
reviewed local configuration, inspect every `.dpkg-dist`, and validate each
service after configuration. Do not blindly accept old files where the new
package requires a security or syntax change.

The clone rehearsal found that Debian 11's optional `sa-compile` package can
fail with the packaged `re2c`; normal SpamAssassin operation still passed lint.
If this recurs, treat it as an optional package issue and document the chosen
resolution.

## 6. Transfer data and settings

Make an initial copy while the old server remains authoritative, followed by a
short final synchronisation during cutover:

- Copy mailbox data preserving ownership, modes, timestamps and dotfiles.
- Recreate or restore mailbox accounts and passwords through the supported
  Dovecot mechanism; keep password databases out of Git.
- Copy reviewed Postfix, Dovecot, OpenDKIM and filtering settings, adapting
  hostname, certificate paths, sockets and interface addresses.
- Import database dumps into new databases with separate restricted users.
- Migrate only reviewed Roundcube configuration. Do not copy obsolete PHP code
  without a vulnerability review.
- Issue or install certificates and verify their permissions.

## 7. Pre-cutover testing

Record every test with date, source and result:

```sh
postfix check
doveconf -n
spamassassin --lint
fail2ban-client status
shorewall check
systemctl --failed
```

Then test IMAPS login, authenticated SMTP submission with TLS, local delivery,
inbound and outbound delivery through multiple external providers, DKIM/SPF/
DMARC headers, spam and virus filtering, and Roundcube login/read/send/
attachments. Test from an external network. Reboot the new host and confirm
all required services return automatically. Confirm Apache and legacy web apps
remain disabled unless explicitly approved.

Do not declare success merely because ports are listening; inspect logs and
queues and test complete mail flows.

## 8. Cutover

1. Announce a short maintenance window.
2. Quiesce web/database writes if those systems are being migrated.
3. Perform final mailbox and configuration synchronisation.
4. Confirm hostname, reverse DNS, clock, certificates, signing and backups.
5. Update MX and mail A/AAAA records, then SPF/DKIM/DMARC as planned. Update
   web records separately if appropriate.
6. Monitor DNS, Postfix queues, delivery logs, Dovecot authentication,
   Fail2Ban and resource usage continuously.
7. Test from an external network and check both old and new queues during DNS
   propagation.

Keep the old host available but prevent it from accepting new mail once DNS
has converged, taking care not to create duplicate delivery.

## 9. Acceptance and rollback decision

Accept only after an observation period with no unexplained delivery failures
or growing queue, reliable IMAP/SMTP authentication, valid TLS and DKIM,
expected filtering, no repeated service/firewall failures, verified new-host
backups, and acceptable CPU, memory, disk and volume usage.

Rollback if delivery, authentication, data, signing or stability is materially
wrong: restore the previous DNS records, retain new mail for reconciliation,
and use the old host only after confirming its state.

## 10. Web decision and cleanup

After mail is stable, decide separately whether to leave Joomla, MediaWiki and
Piwigo offline, rebuild them on a supported separate VM/managed host, or
migrate only reviewed content and databases. Do not re-enable the old
Apache/PHP applications on the mail host.

After the retention period, revoke replaced certificates and keys, remove old
DNS records, delete obsolete snapshots only after backup verification, and
decommission `vm3` last.
