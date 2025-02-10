Hardware Requirements

Minimum

3CPU 4RAM 80GB
Recommended

4CPU 8RAM 160GB
Rent On Hetzner | Rent On OVH
Dependencies Installation

**Install dependencies for building from source**
```
sudo apt update
sudo apt install -y curl git jq lz4 build-essential
```

**Install Go**
```
sudo rm -rf /usr/local/go
curl -L https://go.dev/dl/go1.22.7.linux-amd64.tar.gz | sudo tar -xzf - -C /usr/local
echo 'export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin' >> $HOME/.profile
source .profile
```

Node Installation
**Clone project repository**
```
cd && rm -rf terp-core
git clone https://github.com/terpnetwork/terp-core.git
cd terp-core
git checkout v4.2.2
```

**Build binary**
```
make install
```

**Prepare cosmovisor directories**
```
mkdir -p $HOME/.terp/cosmovisor/genesis/bin
ln -s $HOME/.terp/cosmovisor/genesis $HOME/.terp/cosmovisor/current -f
```

# Copy binary to cosmovisor directory
cp $(which terpd) $HOME/.terp/cosmovisor/genesis/bin

# Set node CLI configuration
terpd config chain-id morocco-1
terpd config keyring-backend file
terpd config node tcp://localhost:26657

# Initialize the node
terpd init "Your Node Name" --chain-id morocco-1

# Download genesis and addrbook files
curl -L https://snapshots.nodejumper.io/terp/genesis.json > $HOME/.terp/config/genesis.json
curl -L https://snapshots.nodejumper.io/terp/addrbook.json > $HOME/.terp/config/addrbook.json

# Set seeds
sed -i -e 's|^seeds *=.*|seeds = "d8256642afae77264bcce1631d51233a9d00249b@terp-mainnet-seed.itrocket.net:13656,5f5cfac5c38506fbb4275c19e87c4107ec48808d@seeds.nodex.one:10410,8542cd7e6bf9d260fef543bc49e59be5a3fa9074@seed.publicnode.com:26656,a81dc3bf1bb1c3837b768eeb82659eecc971890b@terp-mainnet-peer.itrocket.net:13656"|' $HOME/.terp/config/config.toml

# Set minimum gas price
sed -i -e 's|^minimum-gas-prices *=.*|minimum-gas-prices = "0uterp"|' $HOME/.terp/config/app.toml

# Set pruning
sed -i \
  -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "17"|' \
  $HOME/.terp/config/app.toml

# Enable prometheus
sed -i -e 's|^prometheus *=.*|prometheus = true|' $HOME/.terp/config/config.toml

# Download latest chain data snapshot
curl "https://snapshots.nodejumper.io/terp/terp_latest.tar.lz4" | lz4 -dc - | tar -xf - -C "$HOME/.terp"

# Install Cosmovisor
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.7.0

# Create a service
sudo tee /etc/systemd/system/terp.service > /dev/null << EOF
[Unit]
Description=Terp-Network node service
After=network-online.target
[Service]
User=$USER
WorkingDirectory=$HOME/.terp
ExecStart=$(which cosmovisor) run start
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
Environment="DAEMON_HOME=$HOME/.terp"
Environment="DAEMON_NAME=terpd"
Environment="UNSAFE_SKIP_BACKUP=true"
Environment="DAEMON_ALLOW_DOWNLOAD_BINARIES=true"
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable terp.service

# Start the service and check the logs
sudo systemctl start terp.service
sudo journalctl -u terp.service -f --no-hostname -o cat
Secure Server Setup (Optional)

# generate ssh keys, if you don't have them already, DO IT ON YOUR LOCAL MACHINE
ssh-keygen -t rsa

# save the output, we'll use it later on instead of YOUR_PUBLIC_SSH_KEY
cat ~/.ssh/id_rsa.pub
# upgrade system packages
sudo apt update
sudo apt upgrade -y

# add new admin user
sudo adduser admin --disabled-password -q

# upload public ssh key, replace YOUR_PUBLIC_SSH_KEY with the key above
mkdir /home/admin/.ssh
echo "YOUR_PUBLIC_SSH_KEY" >> /home/admin/.ssh/authorized_keys
sudo chown admin: /home/admin/.ssh
sudo chown admin: /home/admin/.ssh/authorized_keys

echo "admin ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers

# disable root login, disable password authentication, use ssh keys only
sudo sed -i 's|^PermitRootLogin .*|PermitRootLogin no|' /etc/ssh/sshd_config
sudo sed -i 's|^ChallengeResponseAuthentication .*|ChallengeResponseAuthentication no|' /etc/ssh/sshd_config
sudo sed -i 's|^#PasswordAuthentication .*|PasswordAuthentication no|' /etc/ssh/sshd_config
sudo sed -i 's|^#PermitEmptyPasswords .*|PermitEmptyPasswords no|' /etc/ssh/sshd_config
sudo sed -i 's|^#PubkeyAuthentication .*|PubkeyAuthentication yes|' /etc/ssh/sshd_config

sudo systemctl restart sshd

# install fail2ban
sudo apt install -y fail2ban

# install and configure firewall
sudo apt install -y ufw
sudo ufw default allow outgoing
sudo ufw default deny incoming
sudo ufw allow ssh
sudo ufw allow 9100
sudo ufw allow 26656

# make sure you expose ALL necessary ports, only after that enable firewall
sudo ufw enable

# make terminal colorful
sudo su - admin
source <(curl -s https://raw.githubusercontent.com/nodejumper-org/cosmos-scripts/master/utils/enable_colorful_bash.sh)

# update servername, if needed, replace YOUR_SERVERNAME with wanted server name
sudo hostnamectl set-hostname YOUR_SERVERNAME

# now you can logout (exit) and login again using ssh admin@YOUR_SERVER_IP
