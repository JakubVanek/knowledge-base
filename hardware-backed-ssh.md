# Hardware-backed SSH credentials

I use two solutions in parallel:
* TPM-backed SSH keys bound to my laptop as a primary solution
* FIDO2-backed SSH keys as a backup

## TPM SSH keys

### Basic setup
The idea here is to have the SSH private key generated inside a TPM (Trusted Platform Module,
ideally version 2.0).  The TPM will only allow you to export the key in an encrypted form that
only it knows how to decrypt and use. This saves space inside the TPM and still
(hopefully?) provides good security.

The integration itself happens via a PKCS11 module. See the following articles for setup steps:
* https://incenp.org/notes/2020/tpm-based-ssh-key.html
* https://www.ledger.com/blog/ssh-with-tpm

The result should be that you can connect to a server using just using the PKCS11 module.
You can query the TPM-backed public key using `ssh-keygen -D /usr/lib/x86_64-linux-gnu/libtpm2_pkcs11.so.1`.
```
ssh -I /usr/lib/x86_64-linux-gnu/libtpm2_pkcs11.so.1 <server>
```

### Reducing the frequency of password prompts

I have the TPM key (intentionally) protected by a PIN. However, in some cases
entering the pin over and over can be frustrating.

#### SSH agent

The SSH agent can cache the PIN for you.

First, extract the public key from the TPM:
```bash
ssh-keygen -D /usr/lib/x86_64-linux-gnu/libtpm2_pkcs11.so.1 >~/.ssh/tpm.pub
```

Then I'd configure the server in `~/.ssh/config` like
```
Host my_server
  HostName example.com
  User root
  IdentitiesOnly yes
  IdentityFile ~/.ssh/tpm
  PKCS11Provider /usr/lib/x86_64-linux-gnu/libtpm2_pkcs11.so.1
```

Before using the key for the first time, run the following command.
It will prompt you for the PIN and save it in memory.
```bash
ssh-add -s /usr/lib/x86_64-linux-gnu/pkcs11/libtpm2_pkcs11.so
```

Finally, SSH logins should not prompt for the PKCS11 PIN.
```
ssh my_server
```

To forget the remembered PIN, run
```bash
ssh-add -e /usr/lib/x86_64-linux-gnu/pkcs11/libtpm2_pkcs11.so
```

#### ControlMaster

SSH can reuse existing connections. This allows you to skip the authentication
step. A relevant resource is https://unix.stackexchange.com/a/244755 .
I am using the following snippet in `~/.ssh/config`:
```
Host *
  ControlMaster auto
  ControlPath ~/.ssh/sockets/%r@%h-%p
  ControlPersist 300
```

## FIDO2 SSH keys

