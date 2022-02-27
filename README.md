# sigstore-git-commit

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

## TODO

So far the CLI interface emulates the bits of gpgsm that git uses to sign and verify commits. I'd
like to add a new mode that can be called in a CI environment to verify the signed identity of the
user. Maybe something like this to accept any user from a trusted domain:

    $ sigstore --validate-email-domain yob.id.au

Or single users:

    $ sigstore --validate-email james@yob.id.au --validate-email bob@example.com

The output should be fairly verbose and clear about the identity found in the
current commit, and if it's trusted then why we trust it. 

## Acknowledgements

The OAuth bits of the code are based on earlier work in https://github.com/sigstore/ruby-sigstore/
