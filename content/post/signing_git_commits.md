---
title: "Signing Git Commits"
date: 2022-07-26T22:53:19+08:00
draft: false
summary: A guide to sign Git commits.
tags: ["Signing", "Git", "Commits", "GitHub", "GPG", "macOS", "Ubuntu"]
categories: ["English"]
---

This post is a guide to sign Git commits.

# macOS

## Homebrew

1. Install `gnupg` and `pinentry-mac`.

    ```shell
    brew install gnupg pinentry-mac
    ```

2. Generate a GPG key. You can also refer to the [GitHub Docuemnt](https://docs.github.com/en/authentication/managing-commit-signature-verification/generating-a-new-gpg-key).

    ```shell
    gpg --full-generate-key
    ```
    1. At the prompt, specify the kind of key you want (e.g. `RSA (sign only)`).
    2. At the prompt, specify the key size (>= 4096) you want (e.g. `4096`).

3. Get the GPG key ID from the output of the following command.

    ```shell
    gpg --list-secret-keys --keyid-format=long
    ```

4. Export the GPG key.

    ```shell
    gpg --armor --export <some GPG key ID>
    ```

5. Add the GPG key to GitHub. You can refer to the [GitHub Docuemnt](https://docs.github.com/en/authentication/managing-commit-signature-verification/adding-a-gpg-key-to-your-github-account).

6. Set `gpg-agent` up.

    ```shell
    echo "pinentry-program $(which pinentry-mac)" >> ~/.gnupg/gpg-agent.conf
    killall gpg-agent
    ```

7. Add Git configurations.

    ```shell
    git config --global gpg.program gpg
    git config --global commit.gpgsign true
    ```

8. Check whether a commit was signed.

    ```shell
    git log --show-signature -1
    ```

## MacPorts

1. Install `gnupg2` with the `pinentry_mac` variant.

    ```shell
    sudo port -N install gnupg2 +pinentry_mac
    ```

    If MacPorts reports that `pinentry-mac` requires a full Xcode installation, install its prebuilt package first, then retry:

    ```shell
    sudo port -N -b install pinentry-mac use_xcode=no
    sudo port -N install gnupg2 +pinentry_mac
    ```

    `-b` allows binary packages only; `use_xcode=no` bypasses the Xcode check for this installation. This requires an available prebuilt package for your macOS version and architecture. If none is available, install a compatible full Xcode to build `pinentry-mac` from source.

2. Follow steps `2` to `5` in the Homebrew section above to generate and add your GPG key to GitHub.

3. Set `pinentry-program` in `~/.gnupg/gpg-agent.conf` to the MacPorts application path, replacing any existing `pinentry-program` line:

    ```text
    pinentry-program /Applications/MacPorts/pinentry-mac.app/Contents/MacOS/pinentry-mac
    ```

    Restart `gpg-agent`:

    ```shell
    killall gpg-agent
    ```

4. Follow steps `7` and `8` in the Homebrew section above to configure Git and check a commit's signature.

# Ubuntu Server (22.04)

1. Install `gnupg`.

    ```shell
    sudo apt install gpg
    ```
2. Follow the above steps (described for macOS) from `2` to `5`.

3. Set the environment variable.

    ```shell
    export GPG_TTY="$(tty)"
    ```
4. Add Git configurations.

    ```shell
    git config --global gpg.program gpg
    git config --global commit.gpgsign true
    ```

5. Check whether a commit was signed.

    ```shell
    git log --show-signature -1
    ```
