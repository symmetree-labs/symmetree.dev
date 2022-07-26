+++
title = "Hardware and asymmetry: what's new in Zerostash 0.5"
date = 2022-07-26
+++

0.5 is a huge milestone for Zerostash, paving the way for some long-term plans, including truly write-only backups. With 0.5, we're half-way there to truly ransomware-resistant backups. In this process, Zerostash 0.5 will also transparently upgrade your existing archives to a more robust encryption scheme that mitigates nonce-reuse and potential [partitioning oracle attacks](https://eprint.iacr.org/2020/1491.pdf).

# Write-only archives

With Zerostash 0.5 you can create a write-only archive on an ordinary storage. While this mode is not useful against an attacker destroying your backups, it will help you make sure they can't read the archive contents without the correct keys. Note that the index is still accessible using your symmetric password, so they'll see all the file names, but not the contents.

This is how you do back up your entire `/`:

    0s keys gen /path/to/stash split_key --user server_backup@symmetree.dev \
                                         --read-keyfile ~/read_key.toml \
                                         --write-keyfile ~/write_key.toml
    0s commit --keyfile ~/write.key.toml /path/to/stash /

# Hardware-based encryption with Yubikeys

Road warriors will appreciate that there's now a way to give a bit more of a personal touch to their backups. A Yubikey configured to perform Challenge-Response HMAC-SHA1 operations _can_ require a touch to decrypt then re-seal the archive. Using challenge-response mode also allows you to easily create a backup Yubikey.

Note, that if you decide that touch is what you want, you will need to pay attention to when `0s` finishes crunching your data, and seals the stash. To set up the Yubikey, consult the [amazing documentation](https://strongbox.reamaze.com/kb/yubikey/how-do-i-create-a-yubikey-protected) by Strongbox. If you just wanted to create an archive of your user directory on an external disk, this is how you do it:

    0s commit --yubikey /mnt/path/to/stash /home/user

Depending on your preferences, you may want to create a keyfile:

    0s keys gen /mnt/path/to/stash yubikey slot2 hmac1 --user road_warrior@symmetree.dev --keyfile home_backup.toml 

# Hardware security with macOS Keychain

macOS users will appreciate that they can configure Keychain to store their passwords. On modern mac laptops, this means your Zerostash credentials are protected by the Secure Enclave. You can use this feature in conjunction with your Yubikey or `split_key` keys, too. If you're adventurous, synchronizing your Keychain with your iCloud account will enable access to your stashes on other fruity devices.

To generate a keyfile with Yubikey that picks your password from Keychain:

    0s keys gen /mnt/path/to/stash yubikey --keychain --keyfile home_backup.toml

To simply run a backup and save the password to Keychain:

    0s keys commit --keychain --user home_backup@symmetree.dev /mnt/path/to/stash /home/user

# Changing password

Up until now, there was now way to change the password of a stash once you create it. This has now *changed*. Right this way, please:

    0s keys change /path/to/stash toml --keyfile home_backup.toml

To explore the full suite of new key operations, you can always consult the helpdesk:

    0s keys --help

# I want to try this, right now!

That's good to hear. You can access the binaries for Linux, Windows, and macOS straight from the [GitHub release](https://github.com/symmetree-labs/zerostash/releases/tag/v0.5.0). You can also use [Homebrew](https://github.com/symmetree-labs/homebrew-tap) and [Nix](https://github.com/symmetree-labs/zerostash#installation-on-nixos) to install a packaged version in your system!

Have fun, and happy hacking!
