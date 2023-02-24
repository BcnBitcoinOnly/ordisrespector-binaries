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

## Install Guix

Start by installing the `guix` package provided by Ubuntu, then update it to the latest stable version with its own update command.

```sh
$ sudo apt install guix
$ guix pull
```

After the second command ends, it will suggest to add the following to lines to your `$HOME/.profile` file, do it:

```sh
# append the following to $HOME/.profile:

export GUIX_PROFILE="/home/marcel/.config/guix/current"
. "$GUIX_PROFILE/etc/profile"
```

Apply the changes by running `source $HOME/.profile`, then run `guix --version`. It will suggest to install a Guix package and add one more line to your `.profile` file.

```sh
$ guix install glibc-locales
```
```sh
# append the following to $HOME/.profile:

export GUIX_LOCPATH="$HOME/.guix-profile/lib/locale"
```

Reload your environment again with `source $HOME/.profile`, now `guix --version` should show no warnings.
At this point you can log out and log in again to your user session so that the `.profile` changes are automatically applied to all your shell sessions instead of just the current one.

## Installing the MacOS SDK (optional)

You can fully skip this section if you don't intend to build MacOS binaries.

Warning: only Bitcoin Core v24, v23 and v22 can do MacOS Guix builds.
Download the appropriate Xcode.xip file for the version of Bitcoin Core you want to build.
To be able to download these files you'll need to use a browser that is logged in into the [Apple developer portal].

| Bitcoin Core Version    | XCode                                                                                            |
|-------------------------|--------------------------------------------------------------------------------------------------|
| Bitcoin Core v24 or v23 | [Xcode_12.2.xip](https://download.developer.apple.com/Developer_Tools/Xcode_12.2/Xcode_12.2.xip) |
| Bitcoin Core v22        | [Xcode_12.1.xip](https://download.developer.apple.com/Developer_Tools/Xcode_12.1/Xcode_12.1.xip) |

Assuming you downloaded `Xcode_12.2.xip` into your `$HOME/Downloads` folder, now install the `git` and `cpio` packages, clone Bitcoin Core's [`apple-sdk-tools`] repository and use its `extract_xcode.py` script to extract the archive.

```sh
$ cd $HOME
$ sudo apt install cpio git
$ git clone https://github.com/bitcoin-core/apple-sdk-tools
$ python3 apple-sdk-tools/extract_xcode.py -f Downloads/Xcode_12.2.xip | cpio -d -i
```

This will create a huge `Xcode.app` folder in your current directory.
Now clone the Bitcoin repository to run another helper script that will take this directory and prepare an "Xcode SDK" from it.
Since we are cloning the bitcoin repo take the opportunity to check out the specific version you want to build.
In this guide we are building Bitcoin Core 24.0.1 so we'll check out the `v24.0.1` tag.

```sh
$ cd $HOME
$ git clone https://github.com/bitcoin/bitcoin
$ cd bitcoin
$ git checkout v24.0.1
$ cd ..
$ ./bitcoin/contrib/macdeploy/gen-sdk $HOME/Xcode.app
```

The last command will generate an `Xcode-12.2-12B45b-extracted-SDK-with-libcxx-headers.tar.gz` in your home directory.
It contains the Apple bits that Bitcoin Core v24 and v23 needs to perform a deterministic MacOS build.
Now create dedicated folder in your home directory for MacOS SDKs and extract this .tar.gz in it:

```sh
$ cd $HOME
$ mkdir MacOS-SDKs
$ mv Xcode-12.2-12B45b-extracted-SDK-with-libcxx-headers.tar.gz MacOS-SDKs
$ cd MacOS-SDKs
$ tar zxf Xcode-12.2-12B45b-extracted-SDK-with-libcxx-headers.tar.gz
```

With this folder structure in place we're ready to do Guix MacOS builds.
If you want you can now delete the `XCode.app` directory and the .xip download to reclaim some 70GB of space in your hard drive.
But save the .tar.gz archive (only 70MB) and keep it close to your heart.

## Building Bitcoin Core

Start by creating a couple of temporary folders in your home directory that the build process will use.

```sh
$ cd $HOME
$ mkdir depends-SOURCES_PATH depends-BASE_PATH
```

If you skipped the MacOS section now clone the Bitcoin Core repository and check out the version you want to build.
As stated elsewhere in the walkthrough we're building Bitcoin Core 24.0.1.

If you already cloned the bitcoin repository and checked out the version you want earlier, simply enter the `bitcoin` directory.

```sh
$ cd $HOME
$ sudo apt install git
$ git clone https://github.com/bitcoin/bitcoin
$ cd bitcoin
$ git checkout v24.0.1
```

Now for the most disrespectful part of the guide, while still inside the bitcoin repository, download the `ordinals-filter.patch` file and apply it to the source tree!

```sh
$ wget https://gist.githubusercontent.com/luke-jr/4c022839584020444915c84bdd825831/raw/555c8a1e1e0143571ad4ff394221573ee37d9a56/filter-ordinals.patch
$ git apply filter-ordinals.patch
```

You can use `git diff` to check the changes you've made to the v24.0.1 codebase. They must look like this:

```sh
$ git diff
diff --git a/src/script/interpreter.cpp b/src/script/interpreter.cpp
index 5f4a1aceb..3eec7373f 100644
--- a/src/script/interpreter.cpp
+++ b/src/script/interpreter.cpp
@@ -479,6 +479,14 @@ bool EvalScript(std::vector<std::vector<unsigned char> >& stack, const CScript&
                     return set_error(serror, SCRIPT_ERR_MINIMALDATA);
                 }
                 stack.push_back(vchPushValue);
+                if ((flags & SCRIPT_VERIFY_DISCOURAGE_UPGRADABLE_NOPS) && opcode == OP_FALSE) {
+                    auto pc_tmp = pc;
+                    opcodetype next_opcode;
+                    valtype dummy_data;
+                    if (script.GetOp(pc_tmp, next_opcode, dummy_data) && next_opcode == OP_IF) {
+                        return set_error(serror, SCRIPT_ERR_DISCOURAGE_UPGRADABLE_NOPS);
+                    }
+                }
             } else if (fExec || (OP_IF <= opcode && opcode <= OP_ENDIF))
             switch (opcode)
             {
```

We're ready to start the build proper.
The Guix build is governed with a series of environment variables that we'll now define.
We start with the mandatory ones.

```sh
$ export SOURCES_PATH="$HOME/depends-SOURCES_PATH"
$ export BASE_CACHE="$HOME/depends-BASE_CACHE"
$ export FORCE_DIRTY_WORKTREE=1
```

This will instruct Guix to use the temporary build directories we created earlier, and to not be scared to build Bitcoin core with "uncommitted changes" (the patch we just applied).

Now, if you did the MacOS section also define this env var so that Guix can find the MacOS SDK:

```sh
$ export SDK_PATH="$HOME/MacOS-SDKs"
```

Lastly, we choose which binaries we want to build.
This is controlled with the `HOSTS` environment variable.
If you don't set it, it will build Bitcoin Core for all supported targets.
The exact targets supported vary for each release, you can find the specific list for 24.0.1 [in this link].

If you skipped the MacOS section it is important to define the HOSTS variable without any Apple targets, otherwise Guix won't start.
In this example we'll tell Guix to only build binaries for Linux x86_64, Linux ARM64 (e.g. for a Raspberry Pi 4) and Apple Silicon (since we went through the pain of preparing the Xcode SDK).
This is also make the build run faster than if it builds all 9 targets.

```sh
$ export HOSTS="x86_64-linux-gnu aarch64-linux-gnu arm64-apple-darwin"
```

...and we're ready to buidl.

```sh
$ ./contrib/guix/guix-build
```

A very long and scary build log will start running through your screen.

Now it'd be a good time to let your machine do the deed and take a break.

## Verifying the result

After the build completes it will have created a `guix-build-24.0.1` directory inside the `bitcoin` project.
In turn, this directory will have a subdirectory for each target you built (in our case `x86_64-linux-gnu`, `aarch64-linux-gnu` and `arm64-apple-darwin`).
We are interested in the contents of `ìnstalled/bin` of each of these subdirectories.
You should find our `bitcoind` binary there along with some other ones such as `bitcoin-cli`.

Use `sha256sum` to calculate the `bitcoind` sha256 hashes and compare them to the ones in the corresponding `SHASUMS256` file.
In this example we should compare our hashes against these lines from the `SHASUMS256-24.0.1.txt` file:

```sh
5a63f7cfa64e7d21005b4fb3956a9192442a4565bd7c03824f5559b9b8499552  bitcoind-24.0.1-aarch64-linux-gnu
f70da220ebda9ba57dea5c531e1fa04122bb1fb3b11e9bf336b8996a448bf0b1  bitcoind-24.0.1-arm64-apple-darwin
d31f48b0a06ff754bdc691b88a091bfac88af24a2d75ebf65383ab52b49155ba  bitcoind-24.0.1-x86_64-linux-gnu
```

## Submit your signature

If you go through all this trouble and get exactly the same bitcoind binaries that I'm distributing I encourage you to GPG sign the corresponding SHASUMS256 file like I did and submit it in an issue.
I'll add your name in the main README.md under the [Build Signers] section and include your signature file in the releases page.

### Other Resources

* Guix binary installation document: [https://guix.gnu.org/manual/en/html_node/Binary-Installation.html](https://guix.gnu.org/manual/en/html_node/Binary-Installation.html)
* Guix README, Bitcoin Core: [https://github.com/bitcoin/bitcoin/blob/master/contrib/guix](https://github.com/bitcoin/bitcoin/blob/master/contrib/guix)
* Guix installation docs, Bitcoin Core: [https://github.com/bitcoin/bitcoin/blob/master/contrib/guix/INSTALL.md](https://github.com/bitcoin/bitcoin/blob/master/contrib/guix/INSTALL.md)
* MacOS SDK extraction guide: [https://github.com/bitcoin/bitcoin/tree/master/contrib/macdeploy](https://github.com/bitcoin/bitcoin/tree/master/contrib/macdeploy)


[Ryzen 7 7700X]: https://www.amd.com/en/products/cpu/amd-ryzen-7-7700
[Apple developer portal]: https://developer.apple.com/
[`apple-sdk-tools`]: https://github.com/bitcoin-core/apple-sdk-tools
[in this link]: https://github.com/bitcoin/bitcoin/tree/v24.0.1/contrib/guix#recognized-environment-variables
[Build Signers]: https://github.com/BcnBitcoinOnly/ordisrespector-binaries#build-signers
