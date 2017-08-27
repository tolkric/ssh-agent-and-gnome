# ssh-agent-and-gnome
Fixes the [Could not add identity "/home/user/.ssh/id_ed25519": communication with agent failed] error

## TLDR
Add the following line to your .bashrc:

    SSH_AUTH_SOCK="$(find /tmp/ -type s 2>/dev/null | grep ssh- | grep agent.)"

Source .bashrc

Test it. This restores ssh-agent functioning including SSO. It also works under dash and should work under most/all other shells.

This does depend on your distributions's configuration of openssh, but it works on debian.

## gpg-agent option
If you'd rather use gpg-agent for ssh then use:

    SSH_AUTH_SOCK="$(find /run/user/ -type s 2>/dev/null | grep gpg-agent.ssh)"

This, too, depends on distribution configuration.

## The do-it-the-right-way[TM] option

If you'd rather do the job properly and have a solution that's independent of your choice of agent and its configuration:

Add to .profile:

    REAL_SSH_AUTH_SOCK="$SSH_AUTH_SOCK"

    export REAL_SSH_AUTH_SOCK

(After .profile is processed, gnome keyring overwrites SSH_AUTH_SOCK and causes your problems)

Add to .bashrc or equivalent for your shell of choice:

    if [ -n "$REAL_SSH_AUTH_SOCK" ]; then SSH_AUTH_SOCK="$REAL_SSH_AUTH_SOCK"; fi

If the only ssh keyring you have is gnome, then this will leave gnome in charge. This is the only option which does that. Again, this is dash-friendly.

## What's the catch?
There is one hole in all of my solutions: you need to start a shell - such as bash - before the fix is applied. If you start a program without using a shell (eg by clicking an icon), then SSH_AUTH_SOCK will still point to gnome keyring and you're back to square one.

# WTF is the problem?

There are 2 bugs in gnome keyring:

1   It doesn't handle the full range of ssh key types. In particular it doesn't handle ed25519 keys. If you try to use such a key, it fails. ssh-add assumes you are using real ssh-agent, so it thinks the problem is a communication failure.

2   It over-writes SSH_AUTH_SOCK to point to gnome keyring. It does this even if SSH_AUTH_SOCK is already set and is therefore in use by another keyring/agent. It still does this even if you follow the instructions to disable the ssh component of gnome keyring. This occurs after processing .profile and before processing shell rc files (like .bashrc).

## Why not use rsa keys instead?
You could, and that may be good enough. I'm a little hypervigilant, so I prefer the most-likely-secure option.

RSA keys have the disadvantage of being vulnerable to factorization attacks, and those are getting rather good.

DSA keys are disabled on v7+ of openssh, and do have some design weakness. We have better choices these days.

ECDSA keys are based on elliptic curve mathematics, so their mathematical weaknesses are not so well understood. Also the algorithm is designed by the NSA. That's the same NSA which back-doored the elliptic curve PRNG, and there are some unexplained constants in the ECDSA algorithm. That smells awfully familiar.

ED25519 is based on ECDSA, but uses a different curve. The two systems use the same mathematics and have the same strength. Theoretically, ED25519 is fractionally faster, but you'll never notice it. The benefit is that if ECDSA is backdoored, then ED25519 isn't. Probably.
