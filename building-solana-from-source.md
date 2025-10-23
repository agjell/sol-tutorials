# Building Agave from source

 1. [Introduction](#introduction)
 2. [Initial installation](#initial-installation)
 3. [Update an existing installation](#update-an-existing-installation)
 4. [All-in-one build command](#all-in-one-build-command) <- Use this for future builds/updates


## Introduction

The Agave client is developed by [Anza](https://www.anza.xyz/). Agave is based on the original Solana client, which is not in use anymore. As of Agave v3 Anza is no longer publishing validator binaries. However, building them from source is fairly easy. Here I provide instructions for building and installing Agave, in addition to performing updates on existing installations. At the end I provide an all-in-one command to make future builds easier.

The tutorial has been tested on a fresh install of Ubuntu 24.04.3 LTS. Please post an issue if something is not working.


## Initial installation

### Install dependencies
```diff
! Perform as user with root privileges
```

Before I can build from source I have to install some dependencies. Installing them requires root privileges.
```bash
sudo apt update && sudo apt install -y \
  git libssl-dev libudev-dev pkg-config zlib1g-dev llvm clang cmake make libprotobuf-dev protobuf-compiler libclang-dev
```

### Install build tools
```diff
! Perform as unprivileged user (for example "sol")
```
For the remaining commands I switch to an unprivileged user, without root access. I usually create the user "sol" to store and run the binaries, but you can name the user whatever you want. I do this by running `sudo adduser sol`. To switch to the user I run `su - sol`.

The next step is to install Rust:
```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```
The installer will provide information about the installation. Press **enter** to proceed.

To make the Agave build script able to find the Cargo binaries (Rust's build system and package manager), my terminal session needs to know where Cargo is. This is ensured through an environment variable, which is automatically added to `~/.profile` during the installation, and is loaded every time I log in. So I can either log out and back in again, or I can run this command:
```bash
source $HOME/.cargo/env
```

### Build and install Agave for the first time

Now for the build. You should first check the officially recommended version in Discord for either [mainnet](https://discord.com/channels/428295358100013066/669406841830244375), [testnet](https://discord.com/channels/428295358100013066/594138785558691840) or [devnet](https://discord.com/channels/428295358100013066/749059399875690557). **Replace the version number in the commands below with the recommendation from Discord.**

I clone the source code from GitHub into my home directory:
```bash
git clone https://github.com/anza-xyz/agave.git --depth 1 --branch v3.0.7 ~/agave-src-v3.0.7
```

Then I run the `cargo-install-all.sh` script to build and install Agave. I assign `~/.local/share/solana/install/releases/3.0.7` as my installation directory, to replicate the directory structure of the pre-built binaries.
```bash
bash ~/agave-src-v3.0.7/scripts/cargo-install-all.sh ~/.local/share/solana/install/releases/3.0.7
```
The script will automatically check for dependencies and download any that are missing. The build process will take some time. When it's done it provides a PATH variable for the binaries. I ignore this, as I make a more useful one in the following step.

When installing the *pre-built* binaries, the installer automatically creates a symbolic link from the installation directory (`*/3.0.7`) to another directory (`*/active_release`). This link can be used in service files and environment variables, so these won't have to be amended on every Agave update. When building from source I have to create that link manually:
```bash
ln --symbolic ~/.local/share/solana/install/releases/3.0.7 ~/.local/share/solana/install/active_release
```

To make sure I can run the binaries from any directory I need to add a PATH variable to `~/.profile`. Note that I use the symbolic link from above here:
```bash
echo 'export PATH=~/.local/share/solana/install/active_release/bin:$PATH' >> ~/.profile
```

### Finishing steps

At this point I usually log out and back in again (to reload `~/.profile`), and run a command to verify that the client is working:
```bash
solana --version
```

The system should respond with the version I just installed. If it doesn't, I retrace my steps and trouble-shoot. If everything is working as expected I delete the source directory:
```bash
rm -rf ~/agave-src-v3.0.7
```
Installation complete! The remaining configuration may be done with the help of my [setup tutorial](https://github.com/agjell/sol-tutorials/blob/master/setting-up-a-solana-devnet-validator.md#configure-solana).


## Update an existing installation
```diff
! Perform as unprivileged user (for example "sol")
```

Updating an existing (self built) Agave installation largely follows the same procedure as the initial install. Below I will repeat the necessary steps from above, but spare you the explanations. For the sake of ease, you may also jump straight to the [all-in-one command](#all-in-one-build-command).

First I clone the new release (find the most recent one on [GitHub](https://github.com/anza-xyz/agave/releases)):
```bash
git clone https://github.com/anza-xyz/agave.git --depth 1 --branch v3.0.7 ~/agave-src-v3.0.7
```

Followed by the build and installation process:
```bash
bash ~/agave-src-v3.0.7/scripts/cargo-install-all.sh ~/.local/share/solana/install/releases/3.0.7
```

Next I replace the symbolic link, pointing it to the new installation. I need to use `--force` and `--no-dereference` to overwrite the old link:
```bash
ln --force --no-dereference --symbolic \
  ~/.local/share/solana/install/releases/3.0.7 ~/.local/share/solana/install/active_release
```

Then I run a command to verify that the client is working:
```bash
solana --version
```

If it replies with the updated version tag I delete the source files:
```bash
rm -rf ~/agave-src-v3.0.7
```

Update complete! After a few days of stable operation I delete the binaries from the previous build.


## All-in-one build command

Lastly, I have combined all the necessary commands to build/update Agave into one. This command presumes I have already installed Rust and the required dependencies. Here I first define an environment variable "TAG" with the version I want to build. This version is then called by all the consecutive commands.

```bash
export TAG=v3.0.7 && \
git clone https://github.com/anza-xyz/agave.git --depth 1 --branch $TAG ~/agave-src-$TAG && \
  bash ~/agave-src-$TAG/scripts/cargo-install-all.sh ~/.local/share/solana/install/releases/$TAG && \
  ln --force --no-dereference --symbolic ~/.local/share/solana/install/releases/$TAG ~/.local/share/solana/install/active_release && \
  solana --version
```

If you are only building binaries for a validator you can append the `--validator-only` flag to `cargo-install-all.sh`. By doing this the build process completes faster, because less binaries are built. Like this:
```bash
export TAG=v3.0.7 && \
git clone https://github.com/anza-xyz/agave.git --depth 1 --branch $TAG ~/agave-src-$TAG && \
  bash ~/agave-src-$TAG/scripts/cargo-install-all.sh --validator-only ~/.local/share/solana/install/releases/$TAG && \
  ln --force --no-dereference --symbolic ~/.local/share/solana/install/releases/$TAG ~/.local/share/solana/install/active_release && \
  solana --version
```

That's it!
