# :warning: DRAFT :warning:

# :warning: THIS DOCUMENT IS NOT FINAL YET :warning:

# :warning: DO NOT MAKE ANY CHANGES TO YOUR NODES BASED ON THIS DOC :warning:

# Setting up a node using Cosmovisor

- [:warning: DRAFT :warning:](#warning-draft-warning)
- [:warning: THIS DOCUMENT IS NOT FINAL YET :warning:](#warning-this-document-is-not-final-yet-warning)
- [:warning: DO NOT MAKE ANY CHANGES TO YOUR NODES BASED ON THIS DOC :warning:](#warning-do-not-make-any-changes-to-your-nodes-based-on-this-doc-warning)
- [Setting up a node using Cosmovisor](#setting-up-a-node-using-cosmovisor)
- [Install Cosmovisor](#install-cosmovisor)
- [Set up a new v1.2 node](#set-up-a-new-v12-node)
- [Migrate a running v1.2 node](#migrate-a-running-v12-node)
- [Prepare a v1.3 node upgrade (Shockwave Alpha)](#prepare-a-v13-node-upgrade-shockwave-alpha)

Cosmovisor is a new process manager for cosmos blockchains. It can make low-downtime upgrades smoother, as validators don't have to manually upgrade binaries during the upgrade, and instead can pre-install new binaries, and Cosmovisor will automatically update them based on on-chain SoftwareUpgrade proposals.

You should review the docs for Cosmovisor located here: [https://docs.cosmos.network/master/run-node/cosmovisor.html](https://docs.cosmos.network/master/run-node/cosmovisor.html)

# Install Cosmovisor

```bash
# Get the Cosmovisor binary
wget -O - "https://github.com/cosmos/cosmos-sdk/releases/download/cosmovisor%2Fv1.1.0/cosmovisor-v1.1.0-linux-amd64.tar.gz" | sudo tar -xz -C /bin cosmovisor
sudo chmod +x /bin/cosmovisor

# Make the necessary directory structure for secretd v1.2
mkdir -p ~/.secretd/cosmovisor/genesis/bin

# Setup environment variables for every future bash session
# This will make sure to setup cosmovisor on you shell in
# in case you'll want to run the cosmovisor command manually
echo "# Setup Secret & Cosmovisor
export SCRT_ENCLAVE_DIR="${HOME}/.secretd/cosmovisor/current/bin"
export LD_LIBRARY_PATH="${LD_LIBRARY_PATH}:${HOME}/.secretd/cosmovisor/current/bin"
export PATH="${PATH}:${HOME}/.secretd/cosmovisor/current/bin"
export DAEMON_NAME=secretd
export DAEMON_HOME="${HOME}"
export DAEMON_ALLOW_DOWNLOAD_BINARIES=false
export DAEMON_LOG_BUFFER_SIZE=512
export DAEMON_RESTART_AFTER_UPGRADE=true
export DAEMON_DATA_BACKUP_DIR="${HOME}"
export UNSAFE_SKIP_BACKUP=true" >> ~/.profile
source ~/.profile

# Setup a systemd service
echo "[Unit]
Description=Cosmovisor Secret Network Node
After=network.target

[Service]
Type=simple
ExecStart=/bin/cosmovisor run
User=${USER}
Restart=on-failure
StartLimitInterval=0
RestartSec=3
LimitNOFILE=65535
LimitMEMLOCK=209715200
Environment=SCRT_ENCLAVE_DIR=${HOME}/.secretd/cosmovisor/current/bin
Environment=LD_LIBRARY_PATH=${LD_LIBRARY_PATH}:${HOME}/.secretd/cosmovisor/current/bin
Environment=DAEMON_NAME=secretd
Environment=DAEMON_HOME=${HOME}
Environment=DAEMON_ALLOW_DOWNLOAD_BINARIES=false
Environment=DAEMON_LOG_BUFFER_SIZE=512
Environment=DAEMON_RESTART_AFTER_UPGRADE=true
Environment=DAEMON_DATA_BACKUP_DIR=${HOME}
Environment=UNSAFE_SKIP_BACKUP=true

[Install]
WantedBy=multi-user.target
" | sudo tee /etc/systemd/system/cosmovisor.service > /dev/null

sudo systemctl daemon-reload
sudo systemctl enable cosmovisor
```

# Set up a new v1.2 node

```bash
# Make the necessary directory structure for v1.2
mkdir -p ~/.secretd/cosmovisor/genesis/bin

# Get v1.2 binaries
wget -P /tmp/ "https://github.com/scrtlabs/SecretNetwork/releases/download/v1.2.2/secretnetwork_v1.2.2_mainnet_amd64.deb"

# Verify v1.2 binaries checksum
echo "1a51d3d9324979ef9a1f56023e458023488b4583bf4587abeed2d1f389aea947 /tmp/secretnetwork_v1.2.2_mainnet_amd64.deb" | sha256sum --check

# Extract v1.2 from archive
dpkg-deb -R /tmp/secretnetwork_v1.2.2_mainnet_amd64.deb /tmp/secretnetwork_v1.2.2_mainnet_amd64.deb.extracted

# Move v1.2 binaries to Cosmovisor's directory
mv /tmp/secretnetwork_v1.2.2_mainnet_amd64.deb.extracted/usr/{local/bin/secretd,lib/librust_cosmwasm_enclave.signed.so,lib/libgo_cosmwasm.so} ~/.secretd/cosmovisor/genesis/bin

# For a query node also move the query enclave
sudo mv /tmp/secretnetwork_v1.2.2_mainnet_amd64.deb.extracted/usr/lib/librust_cosmwasm_query_enclave.signed.so ~/.secretd/cosmovisor/genesis/bin
```

Now the Cosmovisor directory structure should look like this:

```console
$ find ~/.secretd/cosmovisor
/home/ubuntu/.secretd/cosmovisor/
/home/ubuntu/.secretd/cosmovisor/genesis/bin/
/home/ubuntu/.secretd/cosmovisor/genesis/bin/libgo_cosmwasm.so
/home/ubuntu/.secretd/cosmovisor/genesis/bin/librust_cosmwasm_enclave.signed.so
/home/ubuntu/.secretd/cosmovisor/genesis/bin/secretd
```

Now you might want to checkout the [How To Join Secret Network as a Full Node](./node-guides/run-full-node-mainnet.md) guide for syncing a node from height 0 or [Sync with State-Sync](./node-guides/state-sync.md) for syncing a node from a more recent block height.

After making the proper configurations to your new node, launch it using `sudo systemctl start cosmovisor`. You should now see blocks executing in the logs (`journalctl -fu secret-node`).

# Migrate a running v1.2 node

```bash
# Make the necessary directory structure for v1.2
mkdir -p ~/.secretd/cosmovisor/genesis/bin

# Disable the old secret-node systemd service
sudo systemctl stop secret-node
sudo systemctl disable secret-node

# Move v1.2 binaries to Cosmovisor's directory
sudo mv /usr/local/bin/secretd /usr/lib/{libgo_cosmwasm.so,librust_cosmwasm_enclave.signed.so} ~/.secretd/cosmovisor/genesis/bin

# For a query node also move the query enclave
sudo mv /usr/lib/librust_cosmwasm_query_enclave.signed.so ~/.secretd/cosmovisor/genesis/bin

# Launch the Cosmovisor service
sudo systemctl start cosmovisor
```

Now the Cosmovisor directory structure should look like this:

```console
$ find ~/.secretd/cosmovisor
/home/ubuntu/.secretd/cosmovisor/
/home/ubuntu/.secretd/cosmovisor/genesis/bin/
/home/ubuntu/.secretd/cosmovisor/genesis/bin/libgo_cosmwasm.so
/home/ubuntu/.secretd/cosmovisor/genesis/bin/librust_cosmwasm_enclave.signed.so
/home/ubuntu/.secretd/cosmovisor/genesis/bin/secretd
```

To relaunch your node, use `sudo systemctl start cosmovisor`. You should now see blocks executing in the logs (`journalctl -fu secret-node`).

# Prepare a v1.3 node upgrade (Shockwave Alpha)

The "Shockwave Alpha" upgrade is anticipated to be on Wednesday April 27th, 2022 at 2:00PM UTC. Below are instructions to prepare Cosmovisor for automatically upgrading your node to v1.3.

```bash
# Make the necessary directory structure for v1.3
mkdir -p ~/.secretd/cosmovisor/upgrades/v1.3/bin

# Get v1.3 binaries
wget -P /tmp/ "https://github.com/scrtlabs/SecretNetwork/releases/download/v1.3.0/secretnetwork_v1.3.0_mainnet_amd64.deb"

# Verify v1.3 binaries checksum
echo "TODO /tmp/secretnetwork_v1.3.0_mainnet_amd64.deb" | sha256sum --check

# Extract v1.3 from archive
dpkg-deb -R /tmp/secretnetwork_v1.3.0_mainnet_amd64.deb /tmp/secretnetwork_v1.3.0_mainnet_amd64.deb.extracted

# Move v1.3 binaries to Cosmovisor's directory
mv /tmp/secretnetwork_v1.3.0_mainnet_amd64.deb.extracted/usr/{local/bin/secretd,lib/librust_cosmwasm_enclave.signed.so,lib/libgo_cosmwasm.so} ~/.secretd/cosmovisor/upgrades/v1.3/bin

# For a query node also move the query enclave
sudo mv /tmp/secretnetwork_v1.3.0_mainnet_amd64.deb.extracted/usr/lib/librust_cosmwasm_query_enclave.signed.so ~/.secretd/cosmovisor/upgrades/v1.3/bin
```

Now the Cosmovisor directory structure should look like this:

```console
$ find ~/.secretd/cosmovisor
/home/ubuntu/.secretd/cosmovisor/
/home/ubuntu/.secretd/cosmovisor/genesis/bin/
/home/ubuntu/.secretd/cosmovisor/genesis/bin/libgo_cosmwasm.so
/home/ubuntu/.secretd/cosmovisor/genesis/bin/librust_cosmwasm_enclave.signed.so
/home/ubuntu/.secretd/cosmovisor/genesis/bin/secretd
/home/ubuntu/.secretd/cosmovisor/upgrades/v1.3/bin/
/home/ubuntu/.secretd/cosmovisor/upgrades/v1.3/bin/libgo_cosmwasm.so
/home/ubuntu/.secretd/cosmovisor/upgrades/v1.3/bin/librust_cosmwasm_enclave.signed.so
/home/ubuntu/.secretd/cosmovisor/upgrades/v1.3/bin/secretd
```

If set up properly, when the time comes Cosmovisor will perform an automatic v1.2 to v1.3 upgrade.
