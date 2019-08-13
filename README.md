# acme-dns-client

This role installs a local certbot and requests certificates.
It uses the manual challenge where it tries to connect to a DNS server in order to complete the `dns-01` challenge.
The server (where the only implementation is currently [acme-dns-pdns](https://github.com/stuvusIT/acme-dns-pdns)) verifies the client by a client certificate, presented via https.

After running, all unknown renew files are moved away, which means that the default systemd service which renews the certificates can run without errors.

## Requirements

Debian

## Role Variables

| Name                                | Default / Mandatory | Description                                                 |
|:------------------------------------|:-------------------:|:------------------------------------------------------------|
| `acme_client_email`                 | :heavy_check_mark:  | The email to register the Letsencrypt account with          |
| `acme_client_rsa_size`              | `2048`              | Size of the generated RSA key                               |
| `acme_client_staging`               | `false`             | Whether to use the Letsencrypt staging server               |
| `acme_client_key`                   | :heavy_check_mark:  | The private TLS key for the connection to the server        |
| `acme_client_cert`                  | :heavy_check_mark:  | The public TLS key for the connection to the server         |
| `acme_client_ca`                    | :heavy_check_mark:  | The CA of the HTTP server running on the DNS server         |
| `acme_client_server`                | :heavy_check_mark:  | The actual DNS server (`https://` is prefixed automatically |
| `acme_client_certs`                 | `{}`                | Name-SANs mapping of certificates                           |
| `acme_client_hooks`                 | `{}`                | Hooks to execute after request/renewal                      |
| `acme_client_remove_unknown_renews` | `true`              | Whether to remove unknown renew configurations              |

### Certificates

Each certificate is a dict entry.
The key is the name of the certificate (which means the name under `/etc/letsencrypt/live`), but this name is not written into the certificate.
Instead, the value of each dict entry is a list of names to put into the certificate and verify.

### Hooks

The hooks script works by concatenating pieces of bash together.
Each piece is put into `()`, so the pieces shouldn't be able to interfere.
The name of each piece (the key of the hooks dict) is just printed before running the hook.

Unlike other hook systems, there is just one hook for all certificates which might be run multiple times, and which might also run when the certificate that your service needs wasn't even renewed.
That's why there is the `needsUpdate` helper function which checks if a certificate needs to be updated.
`needsUpdate` expects the certificate to be copied somewhere from `/etc/letsencrypt` and checks whether the certs match.
This way, every service can have own certificates with permissions that allow this service to read them.
A simple hook example is:

```bash
if needsUpdate mycert /etc/exim/fullchain.pem; then
  cp "$(certPath mycert fullchain)" /etc/exim/fullchain.pem
  cp "$(certPath mycert privkey)" /etc/exim/privkey.pem
  # Use chmod/chown/… here
  systemctl restart exim4
fi
```

As you can see, `needsUpdate` takes two arguments: The name of the certificate, and the path of the fullchain.pem after copied.

Also, there is `certPath` which just outputs the path of a certificate by name.
The function is implemented, so the certificates can be located at another location (which is not implemented by the role yet).

## Example Playbook

```yml
- hosts: imap
  roles:
    - role: acme-dns-client
      acme_client_email: hostmaster@example.com
      acme_client_server: mydns.local
      acme_client_key: /etc/ssl/private.key
      acme_client_cert: /etc/ssl/public.crt
      acme_client_ca: /etc/ssl/certificates/ca-certificates.crt
      acme_client_certs:
        imap:
          - imap.example.com
          - *.imap.example.com
      acme_client_hooks:
        dovecot: |
            if needsUpdate imap /etc/dovecot/fullchain.pem; then
              cp "$(certPath imap fullchain)" /etc/dovecot/fullchain.pem
              cp "$(certPath imap privkey)" /etc/dovecot/privkey.pem
              systemctl reload dovecot
            fi
```

## License

This work is licensed under a [Creative Commons Attribution-ShareAlike 4.0 International License](http://creativecommons.org/licenses/by-sa/4.0/).

## Author Information

- [Janne Heß](https://github.com/dasJ)
