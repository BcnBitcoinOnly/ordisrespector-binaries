# Ordisrespector Binaries

Ready-made builds of Ordisrespector [Bitcoin Core] binaries.
If you can replace the `bitcoind` binary from your node and restart it, you can run [Ordisrespector].

### Available Versions

Bitcoin Core [24.0.1], [23.1], [22.1] and [0.21.2]

## Download & Cryptographic Verification

1. Download the Bitcoin Core version you want from the [Releases section].
   There are quite a few OSes and architectures to choose from, use this table of the most common ones as reference:

   | Target                     | Suffixes                                          |
   |----------------------------|---------------------------------------------------|
   | Linux x64                  | `*-x86_64-linux-gnu`                              |
   | Linux ARM (Raspberry Pi 4) | `*-aarch64-linux-gnu`                             |
   | MacOS ARM (Apple Silicon)  | `*-arm64-apple-darwin`                            |
   | MacOS x64 (Intel)          | `*-x86_64-apple-darwin` `*-x86_64-apple-darwin18` |
   | Windows x64                | `*-x86_64-w64-mingw32`                            |

   Also download the appropriate `SHA256SUMS.txt` and `SHA256SUMS-oomahq.asc` files for your chosen release and put everything in the same directory.

2. Download [oomahq's GPG public key] and use it to verify the integrity on the SHA256SUMS file.
   You must see a `Good signature from "Unhosted Marcelus #371 <oomahq@twitter.com>" [unknown]` message in the output.
   ```sh
   $ gpg --keyserver hkps://keyserver.ubuntu.com --recv-keys 34BE269FE4268497E45588A884C9BB18153758B4
   gpg: key 84C9BB18153758B4: public key "Unhosted Marcellus #371 <oomahq@twitter.com>" imported
   gpg: Total number processed: 1
   gpg:               imported: 1

   $ gpg --verify SHA256SUMS-23.1-oomahq.asc SHA256SUMS-23.1.txt 
   gpg: Signature made Fri Mar  3 20:56:07 2023 UTC
   gpg:                using EDDSA key 34BE269FE4268497E45588A884C9BB18153758B4
   gpg: Good signature from "Unhosted Marcellus #371 <oomahq@twitter.com>" [unknown]
   gpg: WARNING: This key is not certified with a trusted signature!
   gpg:          There is no indication that the signature belongs to the owner.
   Primary key fingerprint: 34BE 269F E426 8497 E455  88A8 84C9 BB18 1537 58B4
   ```

3. If the signature is OK it means that the SHA256SUMS file has not been tampered.
   Now you can safely use it to check the hash of the binary.
   On Linux, if both files are present in the same directory you can check the hash with this command:

   ```sh
   $ sha256sum --check --ignore-missing SHA256SUMS-23.1.txt 
   bitcoind-23.1-x86_64-linux-gnu: OK
   ```

## Installation

1. Rename the verified binary to `bitcoind` (or `bitcoind.exe` if it's a Windows build).

2. Locate the existing `bitcoind` binary in your node and replace it (on Linux, usually at `/usr/local/bin/bitcoind`).

3. Restart the Bitcoin Core service or reboot your node. Done! Enjoy a clean mempool.

## Build Guide

The binaries have been built using Bitcoin Core's [Guix build system].
Guix builds are deterministic, meaning that anyone can follow the same steps and get the exact same binaries, byte-per-byte.

Documentation of the build process is maintained in a [dedicated document].
In the spirit of the Bitcoin ethos, you are encouraged to build your own binaries and verify that they are the same as I distributed in this repo.
If you do your own deterministic build and do this check yourself I invite you to submit your own GPG signature of the SHA256SUMS file to boost the trust in these binaries.

### Build Signers

* [oomahq] ([pubkey](https://keyserver.ubuntu.com/pks/lookup?op=get&search=0x34be269fe4268497e45588a884c9bb18153758b4)) - 24.0.1, 23.1, 22.1, 0.21.2, all architectures.

## Other Resources

* The Ordisrespector web page, by [Semisol](https://twitter.com/Semisol_Public): [https://ordisrespector.com](https://ordisrespector.com)
* MiniBolt guide to build a patched Bitcoin Core 24.0.1 (non-deterministic, easier than Guix), by [TwoFaktor](https://twitter.com/twofaktor): [https://minibolt.info/guide/bonus/bitcoin/ordisrespector.html](https://minibolt.info/guide/bonus/bitcoin/ordisrespector.html)
* [MYNODE](https://mynodebtc.com/) Ordisrespector patching, official: [https://twitter.com/mynodebtc/status/1626242552776265730](https://twitter.com/mynodebtc/status/1626242552776265730)
* [Umbrel](https://umbrel.com/) Ordisrespector patching guide by [AntePurgatorio](https://twitter.com/AntePurgatorio): [https://github.com/printer-jam/umbrel-ordisrespector](https://github.com/printer-jam/umbrel-ordisrespector)
* [Citadel](https://runcitadel.space/) Ordisrespector patching guide by Zehks: [https://github.com/zehks/citadel-ordisrespector](https://github.com/zehks/citadel-ordisrespector)
* [Start9's EmbassyOS](https://start9.com/) patched binaries by [MegaDisrespecter](https://twitter.com/MDisrespecter): [https://twitter.com/MDisrespecter/status/1628570118790983681](https://twitter.com/MDisrespecter/status/1628570118790983681)
* Ordisrespector Docker images by [MaxMoney21M](https://twitter.com/max_money_21M): [https://github.com/maxmoney21m/ordisrespector](https://github.com/maxmoney21m/ordisrespector)
* Ordisrespector thread #1, by [oomahq]: [https://twitter.com/oomahq/status/1621899175079051264](https://twitter.com/oomahq/status/1621899175079051264)
* Ordisrespector thread #2, by [oomahq]: [https://twitter.com/oomahq/status/1623052780280885253](https://twitter.com/oomahq/status/1623052780280885253)
* Ordisrespector thread #3, by [oomahq]: [https://twitter.com/oomahq/status/1625164770037927938](https://twitter.com/oomahq/status/1625164770037927938)

[Bitcoin Core]: https://bitcoincore.org
[Ordisrespector]: https://twitter.com/oomahq/status/1623052780280885253
[24.0.1]: https://github.com/BcnBitcoinOnly/ordisrespector-binaries/releases/tag/24.0.1
[23.1]: https://github.com/BcnBitcoinOnly/ordisrespector-binaries/releases/tag/23.1
[22.1]: https://github.com/BcnBitcoinOnly/ordisrespector-binaries/releases/tag/22.1
[0.21.2]: https://github.com/BcnBitcoinOnly/ordisrespector-binaries/releases/tag/0.21.2
[Releases section]: https://github.com/BcnBitcoinOnly/ordisrespector-binaries/releases
[oomahq's GPG public key]: https://keyserver.ubuntu.com/pks/lookup?op=get&search=0x34be269fe4268497e45588a884c9bb18153758b4
[Guix build system]: https://github.com/bitcoin/bitcoin/blob/master/contrib/guix/README.md
[dedicated document]: /Guix-Guide.md
[oomahq]: https://twitter.com/oomahq
