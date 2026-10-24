# Mail infrastructure

Ansible configuration for rebuilding and maintaining the Kool4Katz mail
infrastructure on a supported Debian release.

The initial scope is deliberately limited to the mail platform:

- Postfix and Dovecot
- OpenDKIM
- SpamAssassin and ClamAV
- Fail2Ban and Shorewall
- Certbot / Let's Encrypt

Mailbox contents, database dumps, TLS private keys, DKIM private keys and
passwords must not be committed here. Use Ansible Vault or an external secret
manager for secrets, and keep backups separate from this repository.

## Intended workflow

1. Capture and review a sanitised baseline from the existing server.
2. Express stable settings as role defaults, variables and templates.
3. Test against the isolated clone using `--check` first.
4. Build a clean supported-Debian host and apply the roles incrementally.
5. Verify mail flow, authentication, TLS, DKIM, filtering and rollback before
   any DNS cutover.

The current legacy configuration is evidence for the migration, not an
automatic target state. Host-specific values belong in inventory or vaulted
variables.
