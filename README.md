# sigstore-git

An experiment in signing git commits with x509 certificates provided by sigstore.

The [fulcio project](https://github.com/sigstore/fulcio) by
[sigstore](https://www.sigstore.dev/) provides short lifespan (20min) x509
certificates that attest that the user controls a particular OIDC identity. At
this stage GitHub, Google and Microsoft identities are supported.

Assuming you trust the sigstore project to only provide signed certificates to users
who really own the OIDC identity, then commits signed by sigstore/fulcio can reliably
by trusted to have been made by the signed identity.

This prototype is in ruby which isn't particularly portable, but it was a fast way
for me to experiment. Sorry 🤷‍♂️

## Why?

My particular interest is in CI. I have some CI pipelines where I'd like to allowlist
the humans who can execute jobs. By checking the signature of the head commit
on a branch about to run on CI, I can be confident that the branch really was
made by a human I trust to execute code in my CI environment.

Assuming I trust sigstore 🙃

## Usage

For now I assume you have a working ruby environment.

Start by cloning this repo:

    git clone https://github.com/yob/sigstore-git-commit.git
    cd sigstore-git-commit

Start by installing the required dependencies

    gem install clamp openid_connect oa-openid omniauth-openid ruby-openid-apps-discovery launchy faraday_middleware json-jwt

Then configure a git repo to sign commits with this executable:

    git config --local gpg.format x509
    git config --local gpg.x509.program /<full path to git checkout>/sigstore

When committing, add -S to have the commit signed. On first use a browser window will open and you'll
be asked to do an OAuth sign in via GitHub, Google or Microsoft.

Optionally, configure git to sign all commits without the -S flag:

    git config --local commit.gpgsign true

You can see the signature by printing the raw commit:

    $ git cat-file commit HEAD
    tree 929fb311f733b493d8b139b5e3c52fdbab4732e2
    parent 008a69623e3fa408a9596240588c8ab212f1b0e9
    author James Healy <james@yob.id.au> 1645997613 +1100
    committer James Healy <james@yob.id.au> 1645998285 +1100
    gpgsig -----BEGIN SIGNED MESSAGE-----
     MIIH+gYJKoZIhvcNAQcCoIIH6zCCB+cCAQExDzANBglghkgBZQMEAgEFADALBgkq
     hkiG9w0BBwGgggVqMIIDajCCAvCgAwIBAgIUAJhvLyJ3VcikUXBqkQyIbcz4KEcw
     CgYIKoZIzj0EAwMwKjEVMBMGA1UEChMMc2lnc3RvcmUuZGV2MREwDwYDVQQDEwhz
     <...>

On GitHub, the appear like this:

![Unverified on GitHub](/images/github-unverified.png)

The signature is unverified on GitHub because the sigstore root certificate is not in the standard
trusted CA bundles.

Certificates are only valid for 20 mins, so you'll need to do the OAuth dance again if you
need to sign another commit 20+ minutes since the last sign in. That's kind of inconvenient, but
it is what it is.

## OpenSSL

Reference commands for manually verifying a commit signature with openssl. This is not useful to users of
this tool, but it can be helpful during development for verification

"git verify-commit HEAD" will call this program with two bits of data: the
signature (in PKCS7 format) and the signed data from the commit object. Start
by writing both these two two files: git-signature.bin and git-signed-data.bin.

Update the signature to use the PKCS7 markers openssl requires:

    $ cat git-signature.bin | sed "s/SIGNED MESSAGE/PKCS7/" > git-signature.pkcs7

Extract the certificate that signed the data from the PKCS7 signature:

    $ openssl pkcs7 -inform PEM -outform PEM -in git-signature.pkcs7 -print_certs > certs.txt

There's probably multiple certificates in certs.txt - the one with a blank
"subject=" is the signing certificate. Copy it into a new file,
signing-cert.pem

Fetch the fulcio CA certificate:

    $ curl https://raw.githubusercontent.com/sigstore/root-signing/main/repository/repository/targets/fulcio_v1.crt.pem > fulcio_v1.crt.pem

We're ready to verify it:

    $ openssl smime -verify -CAfile fulcio_v1.crt.pem -inform PEM -certfile signing-cert.pem -in git-signature.pkcs7 -content git-signed-data.bin  -purpose any -no_check_time

Most of the above command is somewhat obvious. However, these flags are with discussing.

"-purpose any". Certificates signed by fulcio have a key usage exentsion value
of "Digital Signature". "openssl smime" is unwilling to use certificates with
that purpose to verify smime signatures, so we have to disable that rule.
There's some more info at https://technotes.shemyak.com/posts/smimesign-extended-key-usage-extension-for-openssl-pkcs7-verification/

"-no_check_time" fulcio signed certificates are only valid for 20 minutes. Assuming we're running
these commands outside the 20 minute window, we must tell openssl to ignore the expired time. In
real world verification of signed git commits, we probably want to confirm that the commit date
is within the valid 20 minute period.

## Threat Modelling

Considering some attack scenarios and how to mitigate them.

### Can someone find an expired fulcio issued certificate and matching private key, and create a back-dated commit within the 20 minute certificate valid period?

Yes, this is possible. To protect against it, we need two things:

* in signing mode, we should send proof of the signature to rektor for timestamping
* in verify/check mode, we should check for proof of the signature on rektor and verify the timestamp aligns

### Could a bad actor at sigstore issue a certificate without first receiving a valid OIDC token?

Yes, this is possible and it's hard to mitigate against. Similar to how protecting against this issue in the HTTPS TLS certificate world is hard.

Fulcio does publish all certificates to a transparency log. It's out of scope for this tool, but it'd be possible to monitor that for unauthorized certificates.

Another option might be to validate the records in the transparency log against OIDC token issuing logs from your identity provider. Certificates issued without a matching OIDC token log entry are suspicious.

### One of our users had their account at our OIDC provider hacked. We've recovered the account, can I ensure commits from the compromised period are invalid?

This currently isn't an option, but it would be fairly easy to add. Our check and verify modes would need a way to load the identity and time window that should be explicitly denied as valid.

## TODO

* Add addtional flags to --check mode
  * we already have checking the email domain of the signing identity, and checking the identity provider
  * add check for a single specific user
  * add check for commit email matching the signature identity
* Port to go for portability?
* signing mode should submit signing proof to rektor so we can later verify the signature was made when it
  says it was made
* check mode should support looking for proof (via rektor) that the signature was made at the time it claims
  * this defends against someone finding an expired fulcio issued cert and private key, and using it to create
    a back-dated signed commit that appears valid
  * do we want to do this in verify mode as well? It'll make some git calls slower, but might be worthwhile
* Add visual certificate chain to output of --check

 CN=sigstore, O=sigstore.dev [serial: xxx]
   └─ james@yob.id.au (https://accounts.google.com) [serial: xxx]

## Acknowledgements

The OAuth bits of the code are based on earlier work in https://github.com/sigstore/ruby-sigstore/

## Further Reading

* https://dlorenc.medium.com/should-you-sign-git-commits-f068b07e1b1f
* https://github.com/sigstore/cosign/blob/main/FUN.md
* https://github.com/sigstore/fulcio/blob/main/docs/security-model.md
