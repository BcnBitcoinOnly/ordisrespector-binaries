# Deterministic Build Guide

The following is a step-by-step guide to build your own deterministic Ordisrespector Bitcoin Core.
If you complete these steps you should get exactly the same binaries (byte-per-byte) as distributed in this repository.

## Requirements

* Ubuntu 22.04 (not a hard requirement, but to be able to follow the guide verbatim)
* An Apple Developer account (KYCed, but optional. Only required for MacOS binaries)
* A powerful CPU
* Lots of RAM
* About 100GB of free space for intermediary build artifacts
* A bit of proficiency in Linux
* Patience

For reference, my shiny new [Ryzen 7 7700X] workstation with 32GB of DDR5 RAM and an NVMe SSD takes about 2 and a half hours to complete a full build of the 9 different binaries of Bitcoin Core 24.0.1.
Just so you know what you're getting into.

## Install Guix

Start by installing the `guix`, `build-essentials` and `curl` packages provided by Ubuntu, then update it to the latest stable version with its own update command.

```sh
$ sudo apt install guix build-essential curl
$ guix pull
```

After the second command ends, it will suggest to add the following two lines to your `$HOME/.profile` file, do it:

```
# append the following to $HOME/.profile:

export GUIX_PROFILE="$HOME/.config/guix/current"
. "$GUIX_PROFILE/etc/profile"
```

Apply the changes by running `source $HOME/.profile`, then run `guix --version`. It will suggest to install a Guix package and add one more line to your `.profile` file.

```sh
$ guix install glibc-locales
```
```
# append the following to $HOME/.profile:

export GUIX_LOCPATH="$HOME/.guix-profile/lib/locale"
```

Reload your environment again with `source $HOME/.profile`, now `guix --version` should show no warnings.
At this point you can log out and log in again to your user session so that the `.profile` changes are automatically applied to all your shell sessions instead of just the current one.

## Installing the MacOS SDK (optional)

You can fully skip this section if you don't intend to build MacOS binaries.

**Warning:** only Bitcoin Core v22 and onwards can do deterministic MacOS builds.

First download the appropriate `Xcode.xip` file for the version of Bitcoin Core you want to build.
To be able to download these files you'll need to use a browser that is logged into [Apple's developer portal], otherwise these links won't work.

| Bitcoin Core Version | XCode Version                                                                                    |
|----------------------|--------------------------------------------------------------------------------------------------|
| v24 and v23          | [Xcode_12.2.xip](https://download.developer.apple.com/Developer_Tools/Xcode_12.2/Xcode_12.2.xip) |
| v22                  | [Xcode_12.1.xip](https://download.developer.apple.com/Developer_Tools/Xcode_12.1/Xcode_12.1.xip) |

Assuming you downloaded `Xcode_12.2.xip` into your `$HOME/Downloads` folder, now install the `git` and `cpio` packages, clone Bitcoin Core's [`apple-sdk-tools`] repository and use its `extract_xcode.py` script and `cpio` to extract the archive.

```sh
$ cd $HOME
$ sudo apt install cpio git
$ git clone https://github.com/bitcoin-core/apple-sdk-tools
$ python3 apple-sdk-tools/extract_xcode.py -f Downloads/Xcode_12.2.xip | cpio -d -i
```

This will create a huge `Xcode.app` folder in your home directory.
Now clone the Bitcoin repository to run another helper script that will take this directory and prepare an "Xcode SDK" from it.
Since we are cloning the bitcoin repo take the opportunity to check out the specific version you want to build.
In this guide we are building Bitcoin Core 24.0.1, so we'll check out the `v24.0.1-ordisrespector` branch.

You will notice that we are cloning a fork the official Bitcoin Core repository and checking out a non-standard branch.
Later on we'll explain why we needed to fork the project, and how you can verify how these `-ordisrespector` branches differ exactly from the official tagged releases.

```sh
$ cd $HOME
$ git clone https://github.com/BcnBitcoinOnly/bitcoin
$ cd bitcoin
$ git checkout v24.0.1-ordisrespector
$ cd ..
$ ./bitcoin/contrib/macdeploy/gen-sdk $HOME/Xcode.app
Found Xcode (version: 12.2, build id: 12B45b)
Found MacOSX SDK (version: 11.0, build id: 20A2408)
Creating output .tar.gz file...
Adding MacOSX SDK 11.0 files...
Adding libc++ headers...
Done! Find the resulting gzipped tarball at:
/home/you/Xcode-12.2-12B45b-extracted-SDK-with-libcxx-headers.tar.gz
```

The last command will generate an `Xcode-12.2-12B45b-extracted-SDK-with-libcxx-headers.tar.gz` file in your home directory.
It contains the Apple bits that Bitcoin Core needs to perform deterministic MacOS builds.
Now create a dedicated folder in your home directory for MacOS SDKs and extract this .tar.gz in it:

```sh
$ cd $HOME
$ mkdir MacOS-SDKs
$ mv Xcode-12.2-12B45b-extracted-SDK-with-libcxx-headers.tar.gz MacOS-SDKs
$ cd MacOS-SDKs
$ tar zxf Xcode-12.2-12B45b-extracted-SDK-with-libcxx-headers.tar.gz
```

With this folder structure in place we're ready to do Guix MacOS builds.
If you want you can now delete the `XCode.app` directory and the .xip download to reclaim some 70GB of space in your hard drive.
But save the .tar.gz archive somewhere (only 70MB) and keep it close to your heart in case you ever need it again.

## Building Bitcoin Core

Start by creating a couple of temporary folders in your home directory that the build process will use.

```sh
$ cd $HOME
$ mkdir depends-SOURCES_PATH depends-BASE_PATH
```

If you skipped the MacOS section, now clone the Bitcoin Core repository and check out the version you want to build.
As stated elsewhere in the walkthrough we're building Bitcoin Core 24.0.1.

If you already cloned the bitcoin repository and checked out the version you want in the MacOS section, simply enter the `bitcoin` directory.

```sh
$ cd $HOME
$ sudo apt install git
$ git clone https://github.com/BcnBitcoinOnly/bitcoin
$ cd bitcoin
$ git checkout v24.0.1-ordisrespector
```

At this point we must pause to explain why we are cloning a fork of Bitcoin Core instead of the official repository.
It turns out that the Guix build system is deeply integrated with git, and simply patching the source locally will prompt Guix to erase our local changes when we start the build.
Unless we commit the changes into the git repository Guix won't build them.
And if everyone commits the changes by themselves we'd no longer have a deterministic build.

Therefore, we decided to fork Bitcoin Core to add the [`filter-ordinals.patch`] as a single commit after each of the tagged releases we provide in this repo.

In the following links you can use GitHub's "compare" UI to verify that we only did just that:

| Bitcoin Core Version | Comparison between bitcoin/bitcoin and BcnBitcoinOnly/bitcoin                                                                                                                                            |
|----------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 24.0.1               | [https://github.com/bitcoin/bitcoin/compare/v24.0.1...BcnBitcoinOnly:bitcoin:v24.0.1-ordisrespector](https://github.com/bitcoin/bitcoin/compare/v24.0.1...BcnBitcoinOnly:bitcoin:v24.0.1-ordisrespector) |
| 23.1                 | [https://github.com/bitcoin/bitcoin/compare/v23.1...BcnBitcoinOnly:bitcoin:v23.1-ordisrespector](https://github.com/bitcoin/bitcoin/compare/v23.1...BcnBitcoinOnly:bitcoin:v23.1-ordisrespector)         |
| 22.1                 | [https://github.com/bitcoin/bitcoin/compare/v22.1...BcnBitcoinOnly:bitcoin:v22.1-ordisrespector](https://github.com/bitcoin/bitcoin/compare/v22.1...BcnBitcoinOnly:bitcoin:v22.1-ordisrespector)         |
| 0.21.2               | [https://github.com/bitcoin/bitcoin/compare/v0.21.2...BcnBitcoinOnly:bitcoin:v0.21.2-ordisrespector](https://github.com/bitcoin/bitcoin/compare/v0.21.2...BcnBitcoinOnly:bitcoin:v0.21.2-ordisrespector) |

If you're satisfied with the amendment we made to your chosen tagged release, we're now ready to start the build proper.
The Guix build is governed by a series of environment variables that we'll now define.
We start with the mandatory ones:

```sh
$ export SOURCES_PATH="$HOME/depends-SOURCES_PATH"
$ export BASE_CACHE="$HOME/depends-BASE_CACHE"
```

This will simply instruct Guix to use the temporary build directories we created earlier.

Now, if you want MacOS binaries also define this env var so that Guix can find the MacOS SDK:

```sh
$ export SDK_PATH="$HOME/MacOS-SDKs"
```

Lastly, we choose which flavors of the binaries we want to build.
This is controlled with the `HOSTS` environment variable.
If you don't set it, it will build Bitcoin Core for all supported targets (including Windows, MacOS and 6 different Linux architectures).
The exact targets supported vary for each release, you can find the specific list for 24.0.1 [in this link].

If you skipped the MacOS section it is important to define the `HOSTS` variable without any Apple targets, otherwise Guix won't start.
In this example we'll tell Guix to only build binaries for Linux x86_64, Linux ARM64 (e.g. for a Raspberry Pi 4).
This will also make the build run faster than if it had to build all 9 targets.

```sh
$ export HOSTS="x86_64-linux-gnu aarch64-linux-gnu"
```

...and we're ready to buidl.

```sh
$ ./contrib/guix/guix-build
```

A very long and scary build log will start streaming down your console, and your computer's fans will start going brrr.

Now it'd be a good time to let the machine do the deed and take a break.

## Verifying the result

After the build completes it will have created a `guix-build-275e5239b095` directory inside the `bitcoin` project.
In turn, this directory will have a subdirectory for each target you built (in our case `distsrc-275e5239b095-aarch64-linux-gnu`, `distsrc-275e5239b095-arm64-apple-darwin` and `distsrc-275e5239b095-x86_64-linux-gnu`).
We are interested in the contents of `installed/bitcoin-275e5239b095/bin` of each of these subdirectories.
In there you should find your `bitcoind` binary along with some other ones:

```sh
$ ll guix-build-275e5239b095/distsrc-275e5239b095-x86_64-linux-gnu/installed/bitcoin-275e5239b095/bin/
total 510620
drwxr-xr-x 2 you you      4096 mar  4 12:27 ./
drwxr-xr-x 6 you you      4096 mar  4 12:27 ../
-rwxr-xr-x 1 you you   2412928 mar  4 12:27 bitcoin-cli*
-rwxr-xr-x 1 you you   7306712 mar  4 12:27 bitcoin-cli.dbg*
-rwxr-xr-x 1 you you  15339968 mar  4 12:27 bitcoind*
-rwxr-xr-x 1 you you  85050488 mar  4 12:27 bitcoind.dbg*
-rwxr-xr-x 1 you you  39586776 mar  4 12:27 bitcoin-qt*
-rwxr-xr-x 1 you you 111445552 mar  4 12:27 bitcoin-qt.dbg*
-rwxr-xr-x 1 you you   4112824 mar  4 12:27 bitcoin-tx*
-rwxr-xr-x 1 you you  13497848 mar  4 12:27 bitcoin-tx.dbg*
-rwxr-xr-x 1 you you   2199488 mar  4 12:27 bitcoin-util*
-rwxr-xr-x 1 you you   7766024 mar  4 12:27 bitcoin-util.dbg*
-rwxr-xr-x 1 you you   9686216 mar  4 12:27 bitcoin-wallet*
-rwxr-xr-x 1 you you  42755592 mar  4 12:27 bitcoin-wallet.dbg*
-rwxr-xr-x 1 you you  24893312 mar  4 12:27 test_bitcoin*
-rwxr-xr-x 1 you you 156780072 mar  4 12:27 test_bitcoin.dbg*
```

Use `sha256sum` to calculate the `bitcoind` sha256 hashes and compare them to the ones in the corresponding `SHA256SUMS` file.
In this example we should compare our hashes against these lines from the `SHA256SUMS-24.0.1.txt` file:

```
06076453284836c9b4953ad8e1baae1785fdfee72dfeede73645b50d99ab1e3a  bitcoind-24.0.1-aarch64-linux-gnu
b8de26c038edc6559554e835aa5b360b21df0e885d8778a1254e93f6efc8494c  bitcoind-24.0.1-arm64-apple-darwin
87c800677f4b7cd34bfd899a50022833f8e51a90f6d0bba848ca1a25e4f982e1  bitcoind-24.0.1-x86_64-linux-gnu
```

```sh
$ sha256sum guix-build-275e5239b095/distsrc-275e5239b095-aarch64-linux-gnu/installed/bitcoin-275e5239b095/bin/bitcoind
06076453284836c9b4953ad8e1baae1785fdfee72dfeede73645b50d99ab1e3a  guix-build-275e5239b095/distsrc-275e5239b095-aarch64-linux-gnu/installed/bitcoin-275e5239b095/bin/bitcoind

$ sha256sum guix-build-275e5239b095/distsrc-275e5239b095-arm64-apple-darwin/installed/bitcoin-275e5239b095/bin/bitcoind 
b8de26c038edc6559554e835aa5b360b21df0e885d8778a1254e93f6efc8494c  guix-build-275e5239b095/distsrc-275e5239b095-arm64-apple-darwin/installed/bitcoin-275e5239b095/bin/bitcoind

$ sha256sum guix-build-275e5239b095/distsrc-275e5239b095-x86_64-linux-gnu/installed/bitcoin-275e5239b095/bin/bitcoind
87c800677f4b7cd34bfd899a50022833f8e51a90f6d0bba848ca1a25e4f982e1  guix-build-275e5239b095/distsrc-275e5239b095-x86_64-linux-gnu/installed/bitcoin-275e5239b095/bin/bitcoind
```

They're all exact matches. Great success!
Your three hand-made binaries are byte-per-byte copies of the ones I uploaded in this repo.

## Submit your signature

If you go through all this trouble I encourage you to GPG sign the corresponding SHA256SUMS file (or individual binaries) and submit your signature in an issue.
I'll add your name in the main README.md under the [Build Signers] section and include your signature file in the releases page.

## Other Resources

I've put this guide together from the following sources:

* Guix binary installation document: [https://guix.gnu.org/manual/en/html_node/Binary-Installation.html](https://guix.gnu.org/manual/en/html_node/Binary-Installation.html)
* Guix README, Bitcoin Core: [https://github.com/bitcoin/bitcoin/blob/master/contrib/guix](https://github.com/bitcoin/bitcoin/blob/master/contrib/guix)
* Guix installation docs, Bitcoin Core: [https://github.com/bitcoin/bitcoin/blob/master/contrib/guix/INSTALL.md](https://github.com/bitcoin/bitcoin/blob/master/contrib/guix/INSTALL.md)
* MacOS SDK extraction guide: [https://github.com/bitcoin/bitcoin/tree/master/contrib/macdeploy](https://github.com/bitcoin/bitcoin/tree/master/contrib/macdeploy)


[Ryzen 7 7700X]: https://www.amd.com/en/products/cpu/amd-ryzen-7-7700x
[Apple's developer portal]: https://developer.apple.com/
[`apple-sdk-tools`]: https://github.com/bitcoin-core/apple-sdk-tools
[`filter-ordinals.patch`]: https://gist.github.com/luke-jr/4c022839584020444915c84bdd825831
[in this link]: https://github.com/bitcoin/bitcoin/tree/v24.0.1/contrib/guix#recognized-environment-variables
[Build Signers]: https://github.com/BcnBitcoinOnly/ordisrespector-binaries#build-signers
