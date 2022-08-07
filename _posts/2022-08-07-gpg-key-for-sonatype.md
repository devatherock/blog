---
layout: post
title: "Steps to generate GPG key for Sonatype Nexus"
date: 2022-08-07
---

- Generate GPG key using command `gpg --full-generate-key` with below values:

```text
key type: RSA
key size: 3072
Expiration: none
Real name: <Sonatype username>
Email address: <Sonatype email>
Comment: <blank>
```

- Publish public key to a key server

```shell
gpg --keyserver keyserver.ubuntu.com --send-keys <key-id>
```

- Export secret key ring file with ASCII armor(surrounded by ---BEGIN and END---)

```shell
gpg --armor --export-secret-keys <key-id> > secret-key-ring-file.gpg
```

- Get the secret key ring file value as a one-liner with $ instead of newline character, to add to an environment variable

```shell
cat secret-key-ring-file.gpg | sed ':a; $!N; s|\n|$|; ta; P;D' | sed 's/\$/\\n/g'
```