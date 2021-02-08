# Install SSH Key

[![Build][image-build]][link-build]
[![Windows Server 2019][image-verify-windows-2019]][link-verify-windows-2019]
[![macOS Catalina][image-verify-macos-10.15]][link-verify-macos-10.15]
[![macOS Big Sur][image-verify-macos-11.0]][link-verify-macos-11.0]
[![Ubuntu 20.04][image-verify-ubuntu-20.04]][link-verify-ubuntu-20.04]
[![Ubuntu 18.04][image-verify-ubuntu-18.04]][link-verify-ubuntu-18.04]
[![Ubuntu 16.04][image-verify-ubuntu-16.04]][link-verify-ubuntu-16.04]
[![Release][image-release]][link-release]
[![License][image-license]][link-license]
[![Stars][image-stars]][link-stars]

This action installs SSH key in `~/.ssh`.

Useful for SCP, SFTP, and `rsync` over SSH in deployment script.

**Works on all [virtual environments](https://help.github.com/en/actions/automating-your-workflow-with-github-actions/virtual-environments-for-github-hosted-runners#supported-runners-and-hardware-resources) --**
**Windows Server 2019, macOS Big Sur, macOS Catalina, Ubuntu 20.04, Ubuntu 18.04, and Ubuntu 16.04.**

## Usage

Add your SSH key to your product secrets by clicking `Settings` - `Secrets` - `Add a new secret` beforehand.

**NOTE:** OPENSSH format (key begins with `-----BEGIN OPENSSH PRIVATE KEY-----`) may not work due to OpenSSH version on VM.
Please use PEM format (begins with `-----BEGIN RSA PRIVATE KEY-----`) instead.
In order to convert your key inline to PEM format simply use `ssh-keygen -p -m PEM -f ~/.ssh/id_rsa`.

```yaml
runs-on: ubuntu-latest
steps:
- name: Install SSH key
  uses: shimataro/ssh-key-action@v2
  with:
    key: ${{ secrets.SSH_KEY }}
    name: id_rsa # optional
    known_hosts: ${{ secrets.KNOWN_HOSTS }}
    config: ${{ secrets.CONFIG }} # ssh_config; optional
- name: rsync over ssh
  run: rsync ./foo/ user@remote:bar/
```

See [Workflow syntax for GitHub Actions](https://help.github.com/en/articles/workflow-syntax-for-github-actions) for details.

### Install multiple keys

If you want to install multiple keys, call this action multiple times.
It is useful for port forwarding.

**NOTE:**  When this action is called multiple times, **the contents of `known_hosts` and `config` will be appended**. `key` must be saved as different name, by using `name` option.

```yaml
runs-on: ubuntu-latest
steps:
- name: Install SSH key of bastion
  uses: shimataro/ssh-key-action@v2
  with:
    key: ${{ secrets.SSH_KEY_OF_BASTION }}
    name: id_rsa-bastion
    known_hosts: ${{ secrets.KNOWN_HOSTS_OF_BASTION }}
    config: |
      Host bastion
        HostName xxx.xxx.xxx.xxx
        User user-of-bastion
        IdentityFile ~/.ssh/id_rsa-bastion
- name: Install SSH key of target
  uses: shimataro/ssh-key-action@v2
  with:
    key: ${{ secrets.SSH_KEY_OF_TARGET }}
    name: id_rsa-target
    known_hosts: ${{ secrets.KNOWN_HOSTS_OF_TARGET }} # will be appended to existing .ssh/known_hosts
    config: |                                         # will be appended to existing .ssh/config
      Host target
        HostName yyy.yyy.yyy.yyy
        User user-of-target
        IdentityFile ~/.ssh/id_rsa-target
        ProxyCommand ssh -W %h:%p bastion
- name: SCP via port-forwarding
  run: scp ./foo/ target:bar/
```

## Q&A

### SSH failed even though key has been installed.

Check below:

* `Load key "/HOME/.ssh/id_rsa": invalid format`:
    * OPENSSH format (key begins with `-----BEGIN OPENSSH PRIVATE KEY-----`) may not work.
    * Use PEM format (begins with `-----BEGIN RSA PRIVATE KEY-----`). Convert it from OPENSSH format using `ssh-keygen -p -m PEM -f ~/.ssh/id_rsa`
* `Host key verification failed.`:
    * Set `known_hosts` parameter correctly (use `ssh-keyscan` command).

### How do I use encrypted SSH key?

This action doesn't support encrypted key directly.
Here are some solutions:

* decrypting key beforehand: best bet, and works on any VM
* `sshpass` command: next best bet, but not supported on Windows
* `expect` command: be careful not to expose passphrase to console
* `SSH_ASKPASS` environment variable: might be troublesome

### Which one is the best way for transferring files, "direct SCP/SFTP/rsync" or "SCP/SFTP/rsync via bastion"?

I recommend **rsync via bastion**.

```bash
rsync -e "ssh bastion ssh" ./foo/ target:bar/
```

It has some advantages over other methods:

* "Rsync via bastion" doesn't require to update workflow files and `secrets` even if it is necessary to transfer files to multiple servers.
    * Other methods require to update `known_hosts` if servers have changed.
* Rsync:
    * is fastest of all.
    * does **NOT** break files even if disconnected during transferring.
    * can remove files that don't exist on server.
* SCP is [deprecated by OpenSSH](https://www.openssh.com/txt/release-8.0) due to outdated and inflexible protocol.
* Using bastion is more secure because:
    * it is not necessarily to expose SSH port on servers to public.
        * Address filtering is less effective.
        * Because Azure address range is [very wide](https://help.github.com/en/actions/automating-your-workflow-with-github-actions/virtual-environments-for-github-hosted-runners#ip-addresses-of-github-hosted-runners).
        * And will be updated continuously.
    * if security incident ―e.g., private key leaked― occurs, it's OK just to remove `authorized_keys` on bastion.

## License

The scripts and documentation in this project are released under the [MIT License](LICENSE)

## Changelog

See [CHANGELOG.md](CHANGELOG.md).

[image-build]: https://github.com/shimataro/ssh-key-action/workflows/Build/badge.svg?event=push&branch=v2
[link-build]: https://github.com/shimataro/ssh-key-action/actions?query=workflow%3ABuild
[image-verify-windows-2019]: https://github.com/shimataro/ssh-key-action/workflows/Windows%20Server%202019/badge.svg?event=push&branch=v2
[link-verify-windows-2019]: https://github.com/shimataro/ssh-key-action/actions?query=workflow%3A%22Windows+Server+2019%22
[image-verify-macos-11.0]: https://github.com/shimataro/ssh-key-action/workflows/macOS%20Big%20Sur/badge.svg?event=push&branch=v2
[link-verify-macos-11.0]: https://github.com/shimataro/ssh-key-action/actions?query=workflow%3A%22macOS+Big+Sur%22
[image-verify-macos-10.15]: https://github.com/shimataro/ssh-key-action/workflows/macOS%20Catalina/badge.svg?event=push&branch=v2
[link-verify-macos-10.15]: https://github.com/shimataro/ssh-key-action/actions?query=workflow%3A%22macOS+Catalina%22
[image-verify-ubuntu-20.04]: https://github.com/shimataro/ssh-key-action/workflows/Ubuntu%2020.04/badge.svg?event=push&branch=v2
[link-verify-ubuntu-20.04]: https://github.com/shimataro/ssh-key-action/actions?query=workflow%3A%22Ubuntu+20.04%22
[image-verify-ubuntu-18.04]: https://github.com/shimataro/ssh-key-action/workflows/Ubuntu%2018.04/badge.svg?event=push&branch=v2
[link-verify-ubuntu-18.04]: https://github.com/shimataro/ssh-key-action/actions?query=workflow%3A%22Ubuntu+18.04%22
[image-verify-ubuntu-16.04]: https://github.com/shimataro/ssh-key-action/workflows/Ubuntu%2016.04/badge.svg?event=push&branch=v2
[link-verify-ubuntu-16.04]: https://github.com/shimataro/ssh-key-action/actions?query=workflow%3A%22Ubuntu+16.04%22
[image-release]: https://img.shields.io/github/release/shimataro/ssh-key-action.svg
[link-release]: https://github.com/shimataro/ssh-key-action/releases
[image-license]: https://img.shields.io/github/license/shimataro/ssh-key-action.svg
[link-license]: ./LICENSE
[image-stars]: https://img.shields.io/github/stars/shimataro/ssh-key-action.svg
[link-stars]: https://github.com/shimataro/ssh-key-action/stargazers
