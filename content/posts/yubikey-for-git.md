+++
title = 'Yubikey for Git'
date = 2024-12-15T11:14:31-08:00
draft = false
+++

Security keys such as [Yubikey], [Nitrokey] and [Google's Titan Security
Key][Titan] are great for for keeping cryptographic material off of your device
and improving authentication to cryptographic actions. Unfortunately, these
keys are so versatile that it can be unclear exactly how you should use them
for each application. For Yubikey for instance, you might find yourself trying
to choose between:

[Yubikey]: https://www.yubico.com/
[Nitrokey]: https://www.nitrokey.com/
[Titan]: https://store.google.com/product/titan_security_key

- the [PIV Module],
- the [OpenPGP Module],
- and the [FIDO2 / Webauthn Module]

[PIV Module]: https://developers.yubico.com/PIV/
[PKCS#11]: https://en.wikipedia.org/wiki/PKCS_11
[OpenPGP Module]: https://developers.yubico.com/PGP/
[FIDO2 / Webauthn Module]: https://developers.yubico.com/WebAuthn/ 

The problem then typically becomes how you match that cryptographic interface
(say [PKCS#11] in the case of the PIV Module) to the application in question.
It's usually _possible_ to figure that out and there is an article for every
possible permutation, but the easiest path isn't always clear.

This guide explains the most straight forward approach to using security keys
with [`git`](https://git-scm.com/). I promise -- no PGP keys will be generated
in this process 🙏.

There are two cryptographic operations we need to be concerned with when using
`git`:

- Authentication (pushing and pulling changes)
- Signing commits

We can use SSH keys for both operations so lets take a look at how to create
and use those with security keys.

## Authentication

Authentication to push and pull changes to a remote `git` repository such as
[Github], [Gitlab] or [Bitbucket] typically use SSH keys.

[Github]: https://github.com
[Gitlab]: https://gitlab.com
[Bitbucket]: https://bitbucket.org

In my experience, the simplest way to use SSH Keys with a hardware security key
is via [FIDO2 SSH keys]. Generation is as simple as selecting the special `-sk`
key types (e.g `ecdsa-sk` or `ed25519-sk`) when generating a new key:

[FIDO2 SSH keys]: https://man.openbsd.org/ssh-keygen#FIDO_AUTHENTICATOR

```shell
ssh-keygen -t ed25519-sk -f "$HOME/.ssh/git-auth"
```

Ensure this key is used for your `git` host as follows (using Github as an example):

```
# ~/.ssh/config
Host github.com
  IdentityFile ~/.ssh/git-auth
  User git
```

That's really it. You can copy the contents of e.g `$HOME/.ssh/git-auth.pub`
into your `git` host as just like any other SSH public key. When you push or
pull code you'll be prompted to touch your security key to authenticate.

There is no need to set a passphrase on this SSH key either. The main reason to
set a passphrase is to authenticate access to the SSH private key on the
filesystem. These keys are already authenticated via the security key so a
passphrase is redundant.

It's important to note that, by default, the SSH private key for these [FIDO2
SSH keys] are sort of split in two like a [horcrux]. One part of the private
key is on file (in `$HOME/.ssh/git-auth` in our example) and the other part is
on the security key. As a consequence, you lose either the security key or your
device the SSH key will no longer work. A good security measure, but something
to think about if you're trying to use that same SSH key across multiple
devices.

[horcrux]: https://harrypotter.fandom.com/wiki/Horcrux

## Commit Signing

First, its important to mention that `git` commit signing is _mostly security
theatre_. I think [Harley Watson] has [described this situation
well](https://lobi.to/writes/wacksigning/) so I won't add more to that
conversation. That said, sometimes you gotta jump through hoops for work or
maybe you just like the little green Github badges and that's OK too.

[Harley Watson]: https://lobi.to/

Thankfully, we can use [FIDO2 SSH keys] to sign `git` commits as well! I suggest making
a _different_ SSH key for this purpose:

```shell
ssh-keygen -t ed25519-sk -f "$HOME/.ssh/git-signing"
```

To use the git for signing, modify your git config (globally or on a per repository basis):

```
# .gitconfig
[user]
  signingkey = /path/to/.ssh/git-signing.pub
[gpg]
  format = ssh
[commit]
  gpgsign = true
```

### Why use a different key for signing and authentication?

In short, because [Github] forces you to upload a different key for each
purpose, but they have a good reason! Its important to use a different
cryptographic key for each distinct protocol or context. In both the `git`
commit signing and SSH authentication protocols we're producing signatures over
content. What if we can trick someone into signing content in one text that is
valid in the other? For instance, is it possible for you to structure a commit
such that if its signed it would also be possible to interpret as part of SSH
authentication? If these protocols are well designed, hopefully the answer is
no, but we can remove the risk entirely by using separate keys for each
context (this is called cryptographic domain separation).
