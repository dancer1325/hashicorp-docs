# TODO:
TODO:

TODO:
Start initialization with the default options:

```shell-session
$ vault operator init
```

Initialize, but encrypt the unseal keys with pgp keys:

```shell-session
$ vault operator init \
    -key-shares=3 \
    -key-threshold=2 \
    -pgp-keys="keybase:hashicorp,keybase:jefferai,keybase:sethvargo"
```

Initialize Auto Unseal with a non-default threshold and number of recovery keys, and
encrypt the recovery keys with pgp keys:

```shell-session
$ vault operator init \
    -recovery-shares=7 \
    -recovery-threshold=4 \
    -recovery-pgp-keys="keybase:jeff,keybase:chris,keybase:brian,keybase:calvin,keybase:matthew,keybase:vishal,keybase:nick"
```

Encrypt the initial root token using a pgp key:

```shell-session
$ vault operator init -root-token-pgp-key="keybase:hashicorp"
```