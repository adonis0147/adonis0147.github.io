---
title: "Signing Git Commits"
date: 2022-07-26T22:53:19+08:00
draft: false
summary: A guide to sign Git commits.
tags: ["Signing", "Git", "Commits", "MacOS"]
categories: ["English"]
---

This post is a guide to sign Git commits.

# MacOS

1. Install `gnupg` and `pinentry-mac`.

    ```shell
    brew install gnupg pinentry-mac
    ```

2. Generate a GPG key. You can also follow the [Github Docuemnt](https://docs.github.com/en/authentication/managing-commit-signature-verification/generating-a-new-gpg-key).

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

5. Add the GPG key to Github. You can follow the [Github Docuemnt](https://docs.github.com/en/authentication/managing-commit-signature-verification/adding-a-gpg-key-to-your-github-account).

6. Setting `gpg-agent` up.

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
