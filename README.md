# Ledger Set Up

These instructions require NixOS, or the [Nix](https://nixos.org/nix/) package
manager which you can install on any Linux distribution.

## Obtaining the baking platform

Please clone the
[Tezos Baking Platform](https://gitlab.com/obsidian.systems/tezos-baking-platform) and
check out the `develop` branch, updating submodules. These are the commands to run
in the freshly-cloned repo:

```
$ git clone https://gitlab.com/obsidian.systems/tezos-baking-platform.git
$ cd tezos-baking-platform
$ git checkout develop
$ git submodule sync
$ git submodule update --recursive --init
```

## Ledger setup

Follow only the *Ledger firmware update* and *udev rules* instructions from the
[ledger-app-tezos](https://github.com/obsidiansystems/ledger-app-tezos#instructions-for-nixos)
instructions.

Make sure your Ledger is plugged in and on the app selection screen. Then run
this script to install both the Tezos Baking and Tezos Wallet applications onto
your Ledger:

```
$ ledger/setup_ledger.sh
```

and follow the prompts on your Ledger device. This will require a couple of
confirmation button presses, and entering your PIN twice.

The first time you run this, you will be prompted to download some large files
and manually add them to your Nix store. This is due to a known limitation of
Nix; it is not a fatal error message, just a little additional work. Following
the instructions from the error message will look like this:

```
$ wget http://releases.llvm.org/4.0.0/clang+llvm-4.0.0-x86_64-linux-gnu-ubuntu-16.10.tar.xz
$ nix-store --add-fixed sha256 clang+llvm-4.0.0-x86_64-linux-gnu-ubuntu-16.10.tar.xz
$ wget https://launchpadlibrarian.net/251687888/gcc-arm-none-eabi-5_3-2016q1-20160330-linux.tar.bz2
$ nix-store --add-fixed sha256 gcc-arm-none-eabi-5_3-2016q1-20160330-linux.tar.bz2
```

and then run the script again.

## Running a Zeronet Node

Enter the sandbox shell in the `tezos-baking-platform` working copy's root directory:

```
$ nix-shell -A sandbox \
    --option binary-caches "https://cache.nixos.org/ https://nixcache.reflex-frp.org/" \
    --option binary-cache-public-keys "cache.nixos.org-1:6NCHdD59X431o0gWypbMrAURkbJ16ZPMQFGspcDShjY= ryantrinkle.com-1:JJiAKaRv9mWgpVAz8dwewnZe0AzzEAzPkagE9SP5NWI="
```

And this script will run a node on zeronet:

```
$ scripts/zeronet-node.sh
```

The node needs to remain running while you run all of the `tezos-client`
commands.

## Running the client

In a new terminal, enter a nix shell that has `tezos-client` available:

```
$ nix-shell \
    --option binary-caches "https://cache.nixos.org/ https://nixcache.reflex-frp.org/" \
    --option binary-cache-public-keys "cache.nixos.org-1:6NCHdD59X431o0gWypbMrAURkbJ16ZPMQFGspcDShjY= ryantrinkle.com-1:JJiAKaRv9mWgpVAz8dwewnZe0AzzEAzPkagE9SP5NWI=" \
    --attr client
```

Now jump back to the
[ledger-app-tezos](https://github.com/obsidiansystems/ledger-app-tezos)
instructions. All commands there that use `tezos-client` will need to be run in
this shell.

## Resetting the high water mark

The test network is not so immutable as the real blockchain, and so sometimes
you might want to reset your Ledger Baking state to allow it go back to a lower
block height.

Open the Tezos Baking app on the Ledger, and run this:

```
$ ledger/reset.sh
```

## Running the Ledger Tests

Under `scripts` run `fast-baking-sandbox.sh`:

```
$ scripts/fast-baking-sandbox.sh
```

This is a sandbox environment where the protocol parameters are such that the time between blocks is
3 seconds rather than a minute, which allows baking to happen on the Ledger much sooner during the
process. Within the sandbox nix shell, run:

```
$ scripts/ledger-tests.sh
```

This will perform a number of actions using the Ledger, which should
start out open to the Baking App. Currently, which Ledger is to be used
is hard-coded into the script. This can result in the following error:

```
Error:
  Error:
    No Ledger found for tz1VasatP7zmHDxPeBn97YoSFowXLdsBAdW9
```

If you see this error, please edit the script so that instead of `tz1VasatP7zmHD...`, it instead
refers to the `tz1...` string from `tezos-client list connected ledgers`.

It then goes through the following steps:

  * First, it imports the key.
  * Then, it does a self-delegation, which the Baking App should interpret as a request for baking
    authorization, and sign.
  * Then, it attempts a transaction -- which it will retry and fail at until you switch to the Wallet App.
  * Then, it attempts a reset of HWM -- which it will retry and fail at until you switch back to the
    Baking App.
  * Then, it bakes. At some point, you should see something like this:

    ```
    Injected block BLabDMsNsU3g for my-ledger after BMcko9tfLNju (level 48, slot 0, fitness 00::0000000000000043, operations 0+0+0+0)
    ```

    This means that it baked a block at level `48`, and you should
    also see a `48` (or higher -- whatever was most recently reached)
    displayed on the Ledger (when it's not displaying the baking key,
    which it does most of the time).

If any of these steps don't happen/fail even when you're using the right
app/keep on retrying over and over again, then there might be a problem
with the Ledger app. Try running the commands in `ledger-tests.sh`
one by one to see what might have failed.

## Using Docker for bake monitor

Create docker image (with Nix)

```
$ docker load -i $(nix-build -A bake-central-docker --no-out-link)
$ docker run -p=127.0.0.1:8000:8000 tezos
```

# Running the Bake Monitor via Docker

1) Build the bake monitor Docker image via `nix`.

[You need nix 2. Check with `nix --version`.]

```shell
docker load -i $(nix-build -A bake-central-docker --no-out-link)
```

2) Create a Docker network to connect multiple containers.

```shell
docker network create tezos-bake-network
```

3) Start a Postgres database instance in the network.

```shell
docker pull postgres
docker run --name bake-monitor-db --detach --network tezos-bake-network -e POSTGRES_PASSWORD=secret postgres
```

4) Start the bake monitor Docker container in the same network and connect it to the appropriate database.

```shell
docker run --rm -p=8000:8000 --network tezos-bake-network tezos-bake-monitor --pg-connection='host=bake-monitor-db dbname=postgres user=postgres password=secret'
```

5) Visit the bake monitor at `http://localhost:8000`.
