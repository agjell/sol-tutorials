
# Solana benchmarks

**TLDR:** versions 1.5.11 to 1.5.13 were the highest performers, all scoring an average of over 132,000 transactions per seconds (TPS), with very little variance between the three runs I performed per version. Version 1.5.13 scored ~248% (**!**) higher than v1.5.19, which was the lowest performer. The latter scored an average of 38,154 TPS; a performance drop of ~71% compared to v1.5.13.

 1. [Introduction](#introduction)
 2. [Setup](#setup)
 3. [Results](#results)
 4. [Cheat sheet](#cheat-sheet)

## Introduction

I ran these benchmarks ([from these instructions](https://docs.solana.com/cluster/bench-tps)) after the launch of Solana CLI v1.7.1. Numerous validators reported they were unable to catch up after upgrading, and switching back to v1.7.0 would solve the problem for some. This made me wonder if it was possible to measure performance differences between software versions. As it turned out, it was! And the differences were significant, although not between the versions I expected. I started benchmarking at v1.7.1 and stopped when I got to v1.5.5, at which point I had struggled with very high memory usage since v1.5.10. These earlier software versions would saturate my memory after the first or second run, making it problematic to complete three runs per version (which was the plan). I haven't posted the results of v1.5.5 because the benchmark produced errors during one of the runs. This is a useful find in itself, though. Newer versions seem generally better at managing memory usage.


## Setup

Since I wasn't planning on performing so many tests, I didn't consider my test setup that thoroughly. If I had, I would have tested on Ubuntu, which is the officially recommended distro for Solana. I performed the tests on a secondary validator (my backup), which I was currently experimenting with Fedora on. Testing on Fedora can have some advantages, especially when testing on more recent hardware (like the AMD Zen 3 architecture). Fedora comes with newer kernels with support for more hardware, and offer newer software through their package manager. The disadvantage is that newer software can be less stable. Running the benchmarks on a different operating system than the developers are using is another disadvantage. Anyway, it is what it is.

Every version tested was built with `target-cpu=native` ("znver2" in my case). I ran all the binaries from RAM (tmpfs) in an attempt to exclude SSD performance as a variable. However, it became a variable when testing the versions that caused the highest memory usage, as memory started spilling over into swap. I ran `bench-tps.sh` three consecutive times per build, and rebooted the computer between every build. The test setup was as follows:

 - Hardware
	 - AMD Ryzen 9 3950X (stock)
	 - Gigabyte B550 Aorus Elite V2
	 - Crucial 64 GB DDR4-3600 (16-18-18-38)
 - Software
	 - Fedora 34
	 - Kernel 5.12.9-300.fc34.x86_64
	 - GCC 11.1.1
	 - LLVM v12.0.0
	 - Rustc v1.52.1


## Results

It's worth noting that the margin of error is likely a few thousand TPS. The smaller differences should therefore be ignored. The difference between v1.6.7 and 1.6.8, for instance, is most likely within margin of error and can be considered noise. The interesting finds here are the larger differences. At least to me. The "variance" column displays the difference between lowest and highest score for each respective version.

Version|  Run 1 |  Run 2 |  Run 3 | Average|Variance| Note
:----- | -----: | -----: | -----: | -----: | -----: | :--------------------------------------
1.7.3  |  62417 |  62751 |  60984 |  62051 |  2.8 % |
1.7.2  |  69076 |  65467 |  61830 |  65458 | 10.5 % |
1.7.1  |  68809 |  64800 |  63526 |  65712 |  7.7 % |
1.7.0  |  70616 |  66914 |  64766 |  67432 |  8.3 % | Significant regression from 1.6.x
1.6.14 |  91710 |  90103 |  87758 |  89857 |  4.3 % |
1.6.13 |  96028 |  92618 |  88695 |  92447 |  7.6 % |
1.6.12 |  97274 |  94613 |  91405 |  94431 |  6.0 % |
1.6.11 |  92858 |  91452 |  87618 |  90642 |  5.6 % |
1.6.10 |  97333 |  93641 |  91792 |  94255 |  5.7 % |
1.6.9  |  93111 |  88321 |  85482 |  88971 |  8.2 % |
1.6.8  |  95996 |  89891 |  89343 |  91743 |  6.9 % |
1.6.7  |  97043 |  91970 |  90288 |  93100 |  7.0 % |
1.6.6  |  94915 |  92135 |  91198 |  92749 |  3.9 % |
1.6.5  |  94256 |  89341 |  88652 |  90750 |  5.9 % | Significant regression from 1.6.4
1.6.4  | 129145 | 127152 | 122380 | 126226 |  5.2 % | Significant progression from 1.6.3
1.6.3  | 107981 | 104891 | 102335 | 105069 |  5.2 % | Significant regression from 1.6.2
1.6.2  | 125177 | 120774 | 117028 | 120993 |  6.5 % | Significant progression from 1.6.1
1.6.1  |  39042 |  38105 |  37982 |  38376 |  2.7 % | Significant regression from 1.6.0
1.6.0  |  67424 |  68355 |  68232 |  68004 |  1.4 % | Significant progression from 1.5.19
1.5.19 |  38714 |  38165 |  37581 |  38154 |  2.9 % | **Lowest performer**
1.5.18 |  44656 |  44379 |  43799 |  44278 |  1.9 % |
1.5.17 |  41952 |  42065 |  40766 |  41594 |  3.1 % |
1.5.16 |  46776 |  46186 |  46387 |  46450 |  1.3 % |
1.5.15 |  46540 |  45957 |  45785 |  46094 |  1.6 % | Significant regression from 1.5.14
1.5.14 |  77004 |  76682 |  75812 |  76499 |  1.5 % | Significant regression from 1.5.13
1.5.13 | 133307 | 132757 | 132470 | 132845 |  0.6 % | **Highest performer**
1.5.12 | 132840 | 132659 | 132250 | 132583 |  0.4 % | Lowest variance between tests
1.5.11 | 133401 | 132010 | 130847 | 132086 |  1.9 % |
1.5.10 | 133699 | 134032 | 113442 | 127058 | 15.4 % | High memory usage impacting third run
1.5.9  | 132951 | 130184 |  78604 | 113913 | 40.9 % | High memory usage impacting third run
1.5.8  | 133817 | 133789 | 129643 | 132416 |  3.1 % | High memory usage. but no impact (?)
1.5.7  | 132401 | 132706 | 121857 | 128988 |  8.2 % | High memory usage impacting third run
1.5.6  | 134510 | 120728 | 109883 | 121707 | 18.3 % | High memory usage impacting two runs


## Cheat sheet

I made this cheat sheet for people wanting to benchmark their system.

First you need to install prerequisites, for which you need root privileges. This command is for people running Ubuntu:
```bash
sudo apt update && sudo apt install -y \
  libssl-dev libudev-dev pkg-config zlib1g-dev llvm clang make gcc
```

Then you need to install rustup:
```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source $HOME/.cargo/env
```

After installations are complete, open two more SSH shells (so you have three in total) and perform the following commands in order:

*Shell 1*
```bash
git clone https://github.com/solana-labs/solana.git --branch v1.7.1 ~/solana-src-v1.7.1
cd ~/solana-src-v1.7.1
cargo build --release
NDEBUG=1 ./multinode-demo/setup.sh
NDEBUG=1 ./multinode-demo/faucet.sh
```
Continue when you see `Faucet started` near the bottom of your shell window.

*Shell 2*
```bash
cd ~/solana-src-v1.7.1
NDEBUG=1 ./multinode-demo/bootstrap-validator.sh
```
Continue when you see stuff rolling down the screen faster than you can read it. You'll know what I mean.

*Shell 3 (this starts the benchmark)*
```bash
cd ~/solana-src-v1.7.1
NDEBUG=1 ./multinode-demo/bench-tps.sh
```
It takes some time to run `bench-tps.sh`. When it is done it will present the results; maximum and average transactions per second.
