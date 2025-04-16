# Hippius Storage Miner Deployment

## Table of Contents

- [Overview](#overview)
- [What you need before we start](#what-you-need-before-we-start)
  - [Install ansible](#install-ansible)
  - [Clone the storage miner repo](#clone-the-storage-miner-repo)
  - [Install mitogen (optional)](#optional-install-mitogen-for-faster-ansible-runs)
  - [Server requirements](#server-requirements)
  - [Create SSH key](#create-publicprivate-key-pairs)
  - [SSH Access Setup](#ssh-access-setup)
- [Updating configurations](#updating-configurations)
- [Running the playbooks](#running-the-playbooks)
  - [Main Playbook](#run-the-main-playbook)
  - [Other Playbooks](#other-playbooks)
- [Registering your first node](#registering-your-first-node-on-hippius-network)
- [Registering additional nodes](#registering-additional-nodes-on-hippius-network)
- [Useful commands](#useful-commands)
  - [IPFS](#ipfs)
  - [Hippius](#hippius)
  - [ZFS Pool](#zfs-pool)
- [Common errors](#common-errors)
  - [IPFS not starting](#ipfs-not-starting)
  - [Hippius not connecting](#hippius-not-connecting-to-network)

## Overview
This guide helps you deploy a storage miner using IPFS for decentralised storage, backed by ZFS for reliable disk management and register it on the hippius network

By the end of this setup, you'll have:
- A machine (local or VM) with ansible installed and configured
- A target host fully set up
- A registered storage miner on the hippius network

## What you need before we start
- **A local machine (Ubuntu or WSL2 on Windows) or a small VM running Ubuntu. This is where you will run the setup commands using ansible**

#### Install ansible:
```bash
sudo apt update
sudo apt install python3-pip -y
pip install ansible
```

#### Clone the storage miner repo:
```bash
git clone https://github.com/thenervelab/hippius-storage-miner.git
```

#### (Optional) Install mitogen for faster ansible runs:
```bash
wget https://files.pythonhosted.org/packages/source/m/mitogen/mitogen-0.3.22.tar.gz
tar -xzf mitogen-0.3.22.tar.gz
```
##### Create `ansible.cfg` inside the repo directory and paste:
```ini
[defaults]
strategy_plugins = /your/path/to/mitogen-0.3.22/ansible_mitogen/plugins/strategy
strategy = mitogen_linear
[ssh_connection]
ssh_args = -o ControlMaster=auto -o ControlPersist=2m
```
**IMPORTANT:** Replace `/your/path/to/mitogen-0.3.22/...` to the actual directory

- **Your target hosts running Ubuntu 24.04 (with SSH access)**
#### Server requirements:
To run both the hippius blockchain node and IPFS with ZFS pools efficiently, your server should have:
```
CPU: 4 dedicated cores minimum (8+ recommended) - ZFS and substrate benefit from multithreading
RAM: 16GB minimum (32GB+ recommended) - IPFS and ZFS use large memory caches
Storage:
    - 100GB+ SSD - System OS and chain data
    - 2TB minimum (4TB+ recommended) - ZFS pools for IPFS
        - NVME SSDs or Enterprise SSDs preferred, ZFS mirror or RAID-Z recommended
Network:
    - 1Gbps minimum, with at least 100Mbps sustained throughout
    - Plan for 5-10TB+ of monthly traffic

More CPU and RAM improves chain sync, ARC cache efficiency and IPFS performance under load
```
**NOTE:** Since bittensor mining is a competitive, evolving game, these specs may become outdated at any time. It's your responsibility to research and optimise your infrastructure accordingly

#### Create public/private key pairs:
```bash
ssh-keygen
```

#### Copy public key to your target hosts
```bash
ssh-copy-id user@ip
```

##### Or manually copy public key to the target hosts `~/.ssh/authorized_keys` file

## Updating configurations
Once ansible is installed and we have SSH access to the target hosts, we need to configure some files inside the `hippius-storage-miner/` directory

#### Edit `hosts.yml` to define your target hosts:
```bash
nano inventory/production/hosts.yml
```
Replace `YOUR_VALI_IP` with the IP of your target host and `ansible_user: ubuntu` with a valid user e.g. `root`

## Running the playbooks
Once hosts.yml has been configured we can run the playbook to setup the target host automatically

#### Run the main playbook:
```bash
ansible-playbook -i inventory/production/hosts.yml site.yml -e "hippius_hotkey_mnemonic='word1 word2 word3... word12'"
```
**IMPORTANT:** The playbook will automatically detect and format available disks. Make sure no important data is stored on unused disks before proceeding. Replace `'word1 word2 word3... word12'` with your bittensor hotkey seed phrase

You can also configure this variable (and others like disks, ports, directories etc.) inside `group_vars/all.yml` if you'd rather not have them in your bash history and just run:
```bash
ansible-playbook -i inventory/production/hosts.yml site.yml
```
You can clear your bash history with:
```bash
history -c
```

#### Other playbooks:
After running the main playbook, you usually won't need to run these other commands, but they're here if you ever want to do specific things like updating parts of your miner or checking IDs later on

###### Viewing Node IDs:
```bash
ansible-playbook -i inventory/production/hosts.yml show_node_ids.yml
```

###### Updating only the hippius node:
```bash
ansible-playbook -i inventory/production/hosts.yml site.yml --tags hippius
```

###### Updating only the hippius binary:
```bash
ansible-playbook -i inventory/production/hosts.yml update_hippius.yml
```

###### Injecting HIPS keys:
```bash
ansible-playbook -i inventory/production/hosts.yml inject_keys.yml -e "hippius_hotkey_mnemonic='word1 word2 word3... word12'"
```

###### Updating IPFS storage configuration:
```bash
ansible-playbook -i inventory/production/hosts.yml site.yml --tags ipfs-storage-max
```

## Registering your first node on hippius network
After setting up your storage node, you need to register it on the hippius blockchain to start participating in the network and earn rewards

#### Access the substrate portal:
https://polkadot.js.org/apps/?rpc=wss%3A%2F%2Frpc.hippius.network#/extrinsics

#### Import your bittensor hotkey wallet using the Polkadot{.js} extension
**NOTE:** This is your coldkey on the hippius network

#### Register using the extrinsic pallet `registration` → `registerNodeWithColdkey`:
Change to the following and submit transaction:
```
nodeType: StorageMiner
nodeId: <node id shown on ansible finish>
payInCredits: No
ipfsNodeId: <ipfs node id shown on ansible finish>
```

## Registering additional nodes on hippius network
Once your first node is set up and registered, you can deploy additional nodes that are linked to your first node

#### Create a new wallet using the Polkadot{.js} extension
Copy and securely save the generated seed phrase, this wallet will act as the hotkey for your new node

#### Run the ansible setup on the new target hosts
**NOTE:** During setup, use the new wallet you just created (for `hippius_hotkey_mnemonic=`)

#### Access the substrate portal:
https://polkadot.js.org/apps/?rpc=wss%3A%2F%2Frpc.hippius.network#/extrinsics

#### Add the new wallet as a proxy using extrinisic pallet `proxy` → `addProxy`:
Change to the following and submit transaction:
```
using the selected account: <select your coldkey>
delegate: id
Id: <select your new wallet>
proxyType: NonTransfer
delay: 0
```
**NOTE:** When you click Submit Transaction, uncheck "use a proxy for this call" on the popup page

#### Transfer halpha from your coldkey to hotkey:
In the top menu, go to "Accounts" then choose "Transfer", type in a _very very very_ small number and submit it. This is so it can sign the registration transaction

#### Register using the extrinsic pallet `registration` → `registerNodeWithHotkey`:
Change to the following and submit transaction:
```
using the selected account: <select your hotkey (new wallet)>
coldkey: <select your coldkey>
nodeType: StorageMiner
nodeId: <node id shown on ansible finish>
payInCredits: No
ipfsNodeId: <ipfs node id shown on ansible finish>
```
**NOTE:** Do not use the IDs from your first node! Use the `nodeId` and `ipfsNodeId` generated by the ansible setup on the new host

## Useful commands

#### IPFS

###### Check status:
```bash
systemctl status ipfs
```

###### Restart:
```bash
systemctl restart ipfs
```
###### Check logs:
```bash
journalctl -u ipfs -f -n 100
```

#### Hippius

###### Check status:
```bash
systemctl status hippius
```

###### Restart:
```bash
systemctl restart hippius
```

###### Check logs:
```bash
journalctl -u hippius -f -n 100
```

#### ZFS Pool

###### Check status:
```bash
zpool status ipfs
```

###### List space usage:
```bash
zfs list
```

###### Scrub ZFS pool (recommended monthly):
```bash
zpool scrub ipfs
```

## Common errors

#### IPFS not starting

###### Check permissions:
```bash
ls -la /zfs/ipfs
```

###### Verify ZFS mounts:
```bash
zfs list
```

###### Check IPFS config:
```bash
cat /zfs/ipfs/data/config
```

#### Hippius not connecting to network

###### Check bootnodes configuration
```bash
systemctl cat hippius
```

###### Verify firewall settings:
```bash
ufw status
```

###### Check hippius logs for errors:
```bash
journalctl -u hippius -f -n 100
```
