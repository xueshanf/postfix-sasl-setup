## Configure a Postfix server to allow SASL authentication 

You should adjust your configuration with your own domain, and SASL user database.

All change made to /etc/postfix/main.cf and /etc/postfix/master.cf need a Postfix service reload. 

### Install SASL libraries

```
# apt-get update
# apt-get install sasl2-bin libsasl2-2 libsasl2-modules
```

### Configure Postfix to use SASL and saslauthd

In /etc/postfix/sasl/smtpd.conf:

```
pwcheck_method: saslauthd
mech_list: PLAIN LOGIN
```

Copy the file to /usr/lib/sasl2 directory because libssal uses that location

```
# cp /etc/postfix/sasl/smtpd.conf  /usr/lib/sasl2/smtpd.conf
```

### Configure Postfix service

Enable SASL in Postfix, enforce TLS. In /etc/postfix/main.cf file:

```
smtpd_sasl_auth_enable = yes
smtpd_sasl_path = smtpd
smtpd_sasl_security_options = noanonymous, noplaintext
smtpd_sasl_local_domain =

...
############################################################
# TLS smtpd daemon configuration
############################################################

tls_random_exchange_name = /var/run/postfix/prng_exch

smtpd_tls_CApath                    = /etc/ssl/certs
smtpd_tls_cert_file                 = /etc/ssl/certs/server.pem
smtpd_tls_key_file                  = /etc/ssl/private/server.key
smtpd_tls_mandatory_exclude_ciphers = aNULL

smtpd_tls_loglevel               = 1
smtpd_tls_received_header        = yes
smtpd_tls_security_level         = may
smtpd_tls_session_cache_database = btree:/var/run/postfix/smtpd_scache

```

Allow authenticated users to relay. In /etc/postfix/main.cf:

```
smtpd_recipient_restrictions =
    reject_unknown_recipient_domain,
    permit_sasl_authenticated,
    reject_non_fqdn_recipient,
    reject_unauth_destination
```

### Enbable saslauthd
Edit /etc/default/saslauthd to use saslauthd. The last option  depends on if Postfix is runnign in chroot. Read
the comments in the file for details. 

```
START=yes
NAME="saslauthd"
MECHANISMS="sasldb"
#OPTIONS="-c -m /var/run/saslauthd"
OPTIONS="-c -m /var/spool/postfix/var/run/saslauthd"
```

Start saslauthd

```
# service saslauthd start
```

### Create sasl users and password for sasl authentication

```
# saslpasswd2 -c -u somedomain.com -a smtpauth username
```

Verify:

```
# sasldblistusers2
mail@somedomain.com: userPassword
```
Make sure postfix can read db:

```
# chown postfix:postfix /etc/sasldb2
```

### Test send email

Generate base64 authentication string, for example, to use *plain* authentication method:

```
$ gen-auth plain
Username: username@somedomain.com
Password: password
Auth String: AHNmZW5nQHN0YW5mb3JkLmVkdQBmb29iYXI=
```
You can also use *login* method You will have two base64 encoded authentication string, one for username, one for password. 

Test smtp over TLS:

```
$ openssl s_client -connect somedomain.com:25 -starttls smtp
EHLO somedomain.com
250-somedomain.com
250-PIPELINING
250-SIZE 10240000
250-VRFY
250-ETRN
250-AUTH PLAIN LOGIN
250-ENHANCEDSTATUSCODES
250-8BITMIME
250 DSN
AUTH PLAIN AHNmZW5nQHN0YW5mb3JkLmVkdQBmb29iYXI=
235 2.7.0 Authentication successful
...
mail from: <someuser@somdomain.com>
rcpt to: <someuser@example.com>
data
some message
.

```

### Use an alternate SMTP port
When cloud or ISP providers block outgoing port SMTP port 25, or other well known smtp ports, 
you can configure Postfix to run on a differrent port. Make sure you open firewall rule to allow that port.

Edit /etc/postfix/master.cf and add a line (keep your smtp line if you still want port 25 open):

```
2525 inet  n  -       -       -       -      smtpd
```
