Name
====

dnscrypt-wrapper - A server-side dnscrypt proxy.

(c) 2012-2015 Yecheng Fu <cofyc.jackson at gmail dot com>

[![Build Status](https://travis-ci.org/Cofyc/dnscrypt-wrapper.png?branch=master)](https://travis-ci.org/Cofyc/dnscrypt-wrapper)

Description
===========

This is dnscrypt wrapper (server-side dnscrypt proxy), which helps to
add dnscrypt support to any name resolver.

This software is modified from
[dnscrypt-proxy](https://github.com/jedisct1/dnscrypt-proxy).

Installation
============

Install [libsodium](https://github.com/jedisct1/libsodium) and [libevent](http://libevent.org/) 2 first.

On Linux:

    $ ldconfig # if you install libsodium from source
    $ git clone --recursive git://github.com/Cofyc/dnscrypt-wrapper.git
    $ cd dnscrypt-wrapper
    $ make configure
    $ ./configure
    $ make install

On FreeBSD:

    $ pkg_add -r gmake autoconf
    $ pkg_add -r libevent2
    $ gmake LDFLAGS='-L/usr/local/lib/event2 -L/usr/local/lib' CFLAGS=-I/usr/local/include

On OpenBSD:

    $ pkg_add -r gmake autoconf
    $ pkg_add -r libevent
    $ gmake LDFLAGS='-L/usr/local/lib/' CFLAGS=-I/usr/local/include/

On MacOS:

    $ brew install dnscrypt-wrapper # best recommended

In Docker:

    See https://github.com/jedisct1/dnscrypt-server-docker.

Usage
=====

1) Generate the provider key pair:

    $ dnscrypt-wrapper --gen-provider-keypair

This will create two files in the current directory: `public.key` and
`secret.key`.

This is a long-term key pair that is never supposed to change unless the
secret key is compromised. Make sure that `secret.key` is securely
stored and backuped.

If you forgot to save your provider public key fingerprint:

    $ dnscrypt-wrapper --show-provider-publickey-fingerprint --provider-publickey-file <your-publickey-file>

This will print it out.

2) Generate a time-limited secret key, which will be used to encrypt
and authenticate DNS queries. Also generate a certificate for it:

    $ dnscrypt-wrapper --gen-crypt-keypair --crypt-secretkey-file=1.key
    $ dnscrypt-wrapper --gen-cert-file --crypt-secretkey-file=1.key --provider-cert-file=1.cert

In this example, the time-limited secret key will be saved as `1.key`
and its related certificate as `1.cert` in the current directory.

Time-limited secret keys and certificates can be updated at any time
without requiring clients to update their configuration.

3) Run the program with a given key, a provider name and the most recent certificate:

    # dnscrypt-wrapper --resolver-address=114.114.114.114:53 --listen-address=0.0.0.0:443 \
                       --provider-name=2.dnscrypt-cert.yechengfu.com \
                       --crypt-secretkey-file=1.key --provider-cert-file=1.cert

The provider name can be anything; it doesn't have to be within an existing domain name.
However, it has to start with `2.dnscrypt-cert.`.

When the service is started with the `--provider-cert-file` switch, the
proxy will automatically serve the certificate as a TXT record when a
query for the provider name is received.

As an alternative, the TXT record can be served by a name server for
an actual DNS zone you are authoritative for. In that scenario, the
`--provider-cert-file` option is not required, and instructions for
Unbound and TinyDNS are displayed by the program when generating a
provider certificate.

4) Run dnscrypt-proxy to check if it works:

    # dnscrypt-proxy --local-address=127.0.0.1:55 --resolver-address=127.0.0.1:443 \
                     --provider-name=2.dnscrypt-cert.yechengfu.com \
                     --provider-key=<provider_public_key_fingerprint>
    $ dig -p 55 google.com @127.0.0.1

`<provider_public_key_fingerprint>` is public key fingerprint
generated by `dnscrypt-wrapper --gen-provider-keypair`, which looks
like `4298:5F65:C295:DFAE:2BFB:20AD:5C47:F565:78EB:2404:EF83:198C:85DB:68F1:3E33:E952`.

Optionally, add `-d/--daemonize` flag to run as a daemon.

Run `dnscrypt-wrapper -h` to view command line options.

Running unauthenticated DNS and the dnscrypt service on the same port
=====================================================================

By default, and with the exception of records used for the
certificates, only queries using the DNSCrypt protocol will be
accepted.

If you want to run a service only accessible using DNSCrypt, this is
what you want.

If you want to run a service accessible both with and without
DNSCrypt, what you usually want is to keep the standard DNS port for
the unauthenticated DNS service (53), and use a different port for
DNSCrypt. You don't have to change anything for this either.

However, if you want to run both on the same port, maybe because only
port 53 is reachable on your server, you can add the `-U`
(`--unauthenticated`) switch to the command-line. This is not
recommended.

Key rotation
============

Time-limited keys are bound to expire.

`dnscrypt-proxy` can check if the current key for a given server is
not going to expire soon:

    $ dnscrypt-proxy --resolver-address=127.0.0.1:443 \
                     --provider-name=2.dnscrypt-cert.yechengfu.com \
                     --provider-key=<provider_public_key_fingerprint> \
                     --test=10080

The `--test` option is followed by a "grace margin".

The command will immediately exit after verifying the certificate validity.

The exit code is `0` if a valid certificate can be used, `2` if no valid
certificates can be used, `3` if a timeout occurred, and `4` if a currently
valid certificate is going to expire before the margin.

The margin is always specificied in minutes.

This can be used in a cron tab to trigger an alert before a key is
going to expire.

In order to switch to a fresh new key:

1) Create a new time-limited key (do not change the provider key!) and
its certificate:

    $ dnscrypt-wrapper --gen-crypt-keypair --crypt-secretkey-file=2.key
    $ dnscrypt-wrapper --gen-cert-file --crypt-secretkey-file=2.key --provider-cert-file=2.cert

2) Tell new users to use the new certificate but still accept the old
key until all clients have loaded the new certificate:

    # dnscrypt-wrapper --resolver-address=114.114.114.114:53 --listen-address=0.0.0.0:443 \
                       --provider-name=2.dnscrypt-cert.yechengfu.com \
                       --crypt-secretkey-file=1.key,2.key --provider-cert-file=2.cert

Note that both `1.key` and `2.key` have be specified, in order to
accept both the previous and the current key.

3) Clients automatically check for new certificates every hour. So,
after one hour, the old certificate can be refused, by leaving only
the new one in the configuration:

    # dnscrypt-wrapper --resolver-address=114.114.114.114:53 --listen-address=0.0.0.0:443 \
                       --provider-name=2.dnscrypt-cert.yechengfu.com \
                       --crypt-secretkey-file=2.key --provider-cert-file=2.cert

Please note that on Linux systems (kernel >= 3.9), multiples instances of
`dnscrypt-wrapper` can run at the same time. Therefore, in order to
switch to a new configuration, one can start a new daemon without
killing the previous instance, and only kill the previous instance
after the new one started.

This also allows upgrades with zero downtime.

中文文档
========

注：第三方文档可能未及时与最新版本同步，以 README.md 为准。

- CentOS/Debian/Ubuntu 下编译 dnscrypt-wrapper: http://03k.org/centos-make-dnscrypt-wrapper.html
- dnscrypt-wrapper 使用方法: http://03k.org/dnscrypt-wrapper-usage.html

See also
========

- http://dnscrypt.org/
- https://github.com/jedisct1/dnscrypt-proxy
- https://github.com/Cofyc/dnscrypt-wrapper
