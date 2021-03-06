# Ledger Test Scripts

`fast-baking-sandbox.sh` runs a `nix-shell` command to enter into a Nix sandbox customized with
settings appropriate for Ledger Baking testing by getting the Ledger delegated to bake
relatively quickly.

Within this Nix sandbox, general set up can be done with
`sandbox-nodes.sh`, which sets up a fresh Tezos genesis block and nodes
(killing all existing nodes) but does not run bakers. Bakers can be then
run with `bootstrap-baking.sh` which is not a script in this directory
but a command put in the path by Nix.

`ledger-tests.sh` runs both of those and then runs a series of test commands designed
to test both the baking app and the wallet app. You might have to customize this script to have the
current Ledger path in the import command in this script.

`zeronet-node.sh` creates a new node configuration if it hasn't been created, and then starts
a Zeronet node by querying what to connect to.

`theworks-more.sh` is a general test of Tezos functionality without Ledger included.
