[Sigstore](https://www.sigstore.dev/) can generate short term x509 certificates that are
suitable for signing artifacts like docker images and git commits. A common way
to use Sigstore is in "keyless" mode, where the short term certificate is
linked to an OIDC token. The OIDC token can identify a human (often with an
identity from Google, Microsoft or Github), or sometimes an ontinuous
Integration (CI) provider workflow (like Buildkite, GitHub Actions, Gitlab, or
CircleCI).

[gitsign](https://github.com/sigstore/gitsign) is a program that uses keyless
Sigstore to sign Git commits with an OIDC identity.

Gitsign [v0.6.0 added support for signing with a Buildkite OIDC
token](https://github.com/sigstore/gitsign/releases/tag/v0.6.0), and this is me
experimenting with it.

## Why

Buildkite pipelines are often used to create new commits that are pushed back
to a Git repository. Common use cases include:

* a scheduled build that updates a repository in some
* GitOps: a code pipeline that builds a new artifact (like a docker image) and
  then triggers a deploy of the new code with a git commit

## How it works

from v0.6.0 gitsign looks for the SIGSTORE_ID_TOKEN environment variable. If
found, the OIDC token in the value is used as the identity to fetch an x509
certificate and sign the commit.


## What does the certificate look like?

Git commits signed with an x509 certificate can have the cert printed like this. A few noteworthy things it includes:

* the subject Alternative Name is the Buildkite Pipeline URL
* the issuer identity is https://agent.buildkite.com

Also, the certificate is signed by sigstore.dev. This Certificate Authority is
**NOT** one that's trusted by defualt in any browser or operating system.

```
$ git cat-file commit HEAD | sed -n '/BEGIN/, /END/p' | sed 's/^ //g' | sed 's/gpgsig //g' | sed 's/SIGNED MESSAGE/PKCS7/g' | openssl pkcs7 -print -print_certs -text

...
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            0b:1e:1a:e3:c8:1d:39:12:64:18:64:aa:0a:ab:85:ce:52:e1:aa:2f
        Signature Algorithm: ecdsa-with-SHA384
        Issuer: O=sigstore.dev, CN=sigstore-intermediate
        Validity
            Not Before: May 27 12:17:56 2023 GMT
            Not After : May 27 12:27:56 2023 GMT
        Subject:
        Subject Public Key Info:
            Public Key Algorithm: id-ecPublicKey
                Public-Key: (256 bit)
                pub:
                    04:60:90:72:34:1a:49:35:e6:26:84:80:8f:0d:02:
                    59:38:b3:d6:e0:bd:2d:a6:72:73:5b:ea:34:ba:45:
                    4a:e6:0a:c2:e0:cf:04:d3:07:fd:50:05:0e:2a:a9:
                    a1:17:2e:c6:ad:b1:a4:30:63:af:80:1c:4d:2a:67:
                    ef:59:3d:30:12
                ASN1 OID: prime256v1
                NIST CURVE: P-256
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature
            X509v3 Extended Key Usage:
                Code Signing
            X509v3 Subject Key Identifier:
                57:18:0D:CA:B1:2B:AE:2B:65:86:78:90:55:98:B2:80:2D:00:28:3A
            X509v3 Authority Key Identifier:
                DF:D3:E9:CF:56:24:11:96:F9:A8:D8:E9:28:55:A2:C6:2E:18:64:3F
            X509v3 Subject Alternative Name: critical
                URI:https://buildkite.com/yob-opensource/oidc-signing-experiment
            1.3.6.1.4.1.57264.1.1:
                https://agent.buildkite.com
            1.3.6.1.4.1.57264.1.8:
                ..https://agent.buildkite.com
            CT Precertificate SCTs:
                Signed Certificate Timestamp:
                    Version   : v1 (0x0)
                    Log ID    : DD:3D:30:6A:C6:C7:11:32:63:19:1E:1C:99:67:37:02:
                                A2:4A:5E:B8:DE:3C:AD:FF:87:8A:72:80:2F:29:EE:8E
                    Timestamp : May 27 12:17:56.918 2023 GMT
                    Extensions: none
                    Signature : ecdsa-with-SHA256
                                30:45:02:20:18:E5:5E:D5:33:D3:45:84:48:92:AE:30:
                                5B:C4:CA:6B:6B:A6:FF:58:5A:AC:E0:98:C9:5C:47:EC:
                                78:20:CD:62:02:21:00:8A:A9:C5:D0:8E:F8:04:3F:C3:
                                D1:86:0C:89:90:7D:0D:25:3F:39:E0:35:6A:1C:1D:E9:
                                51:93:04:15:FA:D2:D5
    Signature Algorithm: ecdsa-with-SHA384
    Signature Value:
        30:65:02:30:72:e9:30:f1:fe:79:0b:cd:82:93:fc:54:57:35:
        55:2e:0b:18:fe:59:9a:78:16:7f:b4:2f:a1:a4:43:2d:d2:f4:
        6d:03:3d:35:c9:bb:a4:57:c8:58:fc:5f:d0:86:ff:b3:02:31:
        00:bc:86:6d:6b:57:b8:08:11:7b:ef:df:02:a1:5f:11:28:71:
        a1:fc:6c:16:56:71:78:a1:b2:e5:1b:66:8d:e8:ca:73:08:64:
        ae:6a:33:76:bb:1d:c4:a7:78:58:33:65:7b
...
```

## Verifying a commit was created in a Buildkite build

To verify that a commit was created in a specific Buildkite pipeline, use:

* the Buildkite Pipeline URL as the certificate identity
* "https://agent.buildkite.com" as the OIDC issuer

```
$ gitsign verify --certificate-identity=https://buildkite.com/yob-opensource/oidc-signing-experiment --certificate-oidc-issuer=https://agent.buildkite.com HEAD
tlog index: 21846532
gitsign: Signature made using certificate ID 0x8cd9e267dcc37c109a5c1f1811ad1608c692465c | CN=sigstore-intermediate,O=sigstore.dev
gitsign: Good signature from [https://buildkite.com/yob-opensource/oidc-signing-experiment](https://agent.buildkite.com)
Validated Git signature: true
Validated Rekor entry: true
Validated Certificate claims: true
```

With the wrong signture, the output looks like:

```
$ gitsign verify --certificate-identity=https://buildkite.com/yob-opensource/oidc-signing-experiment --certificate-oidc-issuer=https://agent.buildkite.com HEAD
Error: none of the expected identities matched what was in the certificate, got subjects [joe@example.com] with issuer https://accounts.google.com
```

With no signature the output looks like:

```
$ gitsign verify --certificate-identity=https://buildkite.com/yob-opensource/oidc-signing-experiment --certificate-oidc-issuer=https://agent.buildkite.com HEAD
Error: unsupported signature type
```
