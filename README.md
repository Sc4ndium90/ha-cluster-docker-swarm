# What's my project?
This repository is a full step by step setup of my upcomming high-availability cluster for my homelab dedicated for hosting Docker containers with Docker Swarm, MicroCeph and MariaDB Galera Cluster. The whole setup also host a Traefik router and Crowdsec for web apps, including monitoring with Grafana and Prometheus.

## Why?
I currently have a 5 nodes Proxmox cluster with no high-availability at all levels, which means that any updates requiring a restart of the node, or any failure of a node implies a loss of availability for any apps hosted on the node with no backups. Also, 3 of the nodes in the Proxmox cluster are not powerfull enough to support recent OSes like Windows 11 or Windows Server when making proof of concepts. This project will make those nodes more useful than before. Also, each time I need a new app, I'll have to make a new virtual machine, installing Docker and add the entry to Traefik by hand. This setup is time consumming and we can loose ourselves in a mess like that (which is also not well documented). Most of the apps hosted are websites or web apps on Docker, with Compose files. There is no need for multiple replicas of a website yet, but this setup will allow this if needed.

## How?
Before breaking all my setup and loose time with my current hardware, I'll virtualize the project in VMWare Workstation 25H2 on my desktop to represent realistically. The nodes I'll use for this project are Dell Wyse 5070 with a 4 core Intel Pentium CPU, 16GB of DDR4 memory and a 256GB SSD. My desktop doesn't have that much RAM and CPU, so I'll stick with 2vCPU, 8GB of memory and 256GB of storage running Ubuntu 24.04 LTS. They will be called `docker-eva`, `docker-asuka` and `docker-rei`. To represent the homelab network, I'll use a MikroTik CHR router, with the first interface on the NAT network of VMWare, and a LAN Segment network on the second interface. The LAN network on the lab will be the network subnet `10.1.1.0/24`, with the gateway on `10.1.1.1`, set the DNS server on `1.1.1.1`. I've also set a Virtual IP later in this documentation on `10.1.1.10` for all three nodes with keepalived. 

Here a quick layout of the environment :
*Coming soon*

To be able to copy-paste configurations and run commands, I'll forward in the firewall the SSH port for each node (`docker-eva` will be port 22 to 22, `docker-asuka` port 222 to 22, and `docker-rei` port 2222 to 22).

# Setup of Ubuntu 24.04 LTS
During the setup of Ubuntu, I'll have to make some modifications :
1. Set the interface to static with the dedicated IPv4 with the DNS and gateway
2. Set the hostname of the node.
3. Expand the LVM to its full capacity (Ubuntu only use approx. 30GB of storage for the LVM group).
4. Install Ubuntu with minimal tools.

During the setup on VMWare, I didn't put the right amount of disk space. In this case, there are some commands to run to expand the LVM :
1. Use `cfdisk` to extend `/dev/sda3` (the LVM partition).
2. Run `sudo lvextend -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv`
3. Run `sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv`

# Docker Swarm
We'll start first with Docker Swarm, as it's the quickest and easiest to do. The following steps are given by Docker themselves from their documentation [here](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository) and [here](https://docs.docker.com/engine/install/linux-postinstall)

First, install the Docker repository:
```sh
# Add Docker's official GPG key:
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
```

Then install the packages
```sh
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Finally, add the current user to the docker group and restart the SSH session
```sh
sudo usermod -aG docker $USER
```

Once it is installed on each node, we can initiate the Swarm cluster on the first node (`docker-eva`) with `docker swarm init`. It'll output the following command with the right token and IP from your node. Note that if there's more than one IP on the network interface or multiple network interfaces, you need to specify the IP with `--advertise-addr`. The documentation of this command can be found [here](https://docs.docker.com/reference/cli/docker/swarm/init/). With the given command, apply it on the other two nodes (`docker-asuka` and `docker-rei`)
```sh
docker swarm join --token SWMTKN-1-32urqmbj3ce02el6bbgtlefgk0mjbf17rat5btac2blpoup9f0-5ddwpb9j3g76rfauparnnjleu 10.1.1.11:2377
```

# MicroCeph
MicroCeph is a light version of Ceph. With some command lines, the cluster can be up in minutes and will be enough for a small size cluster. The official documentation of MicroCeph is available [here](https://canonical-microceph.readthedocs-hosted.com/stable/). It is provided by Canonical on Snap:
```sh
sudo snap install microceph
sudo snap refresh --hold microceph
```

Once installed, on the first node (`docker-eva`), bootstrap the cluster with `sudo microceph bootstrap`, then check with `sudo microceph status` that the cluster is initiated. Then, create the tokens for each nodes we'll have to add :
```sh
sudo microceph cluster add docker-asuka
sudo microceph cluster add docker-rei
```

On each other node, use `sudo microceph cluster join <TOKEN>` with the token created for each host, then create an OSD (a 'disk') on each node with `sudo microceph disk add loop,150G,1`. You can check that the cluster is synced and healthy with `sudo microceph.ceph status`.
```sh
cluster:
    id:     be9fd6a9-7bfc-4e9a-9d6d-3daa1d7be688
    health: HEALTH_OK

  services:
    mon: 3 daemons, quorum docker-eva,docker-asuka,docker-rei (age 106m)
    mgr: docker-eva(active, since 106m), standbys: docker-rei, docker-asuka
    mds: 1/1 daemons up, 2 standby
    osd: 3 osds: 3 up (since 106m), 3 in (since 24h)

  data:
    volumes: 1/1 healthy
    pools:   3 pools, 145 pgs
    objects: 3.47k objects, 133 MiB
    usage:   743 MiB used, 449 GiB / 450 GiB avail
    pgs:     145 active+clean
```
If the cluster is healthy, you can create the volume 'docker' with `sudo microceph ceph fs volume create docker`

## Setup CephFS mount
To store data on the Ceph filesystem, we need to mount the storage. For that, on each node, we have to install `ceph-commons` with APT, then create the mount point `/etc/cephfs`.

The mounting requires some MicroCeph configs files: we will make a symbolic link to them:
```sh
sudo ln -sf /var/snap/microceph/current/conf/ceph.client.admin.keyring /etc/ceph/ceph.client.admin.keyring
sudo ln -sf /var/snap/microceph/current/conf/ceph.keyring /etc/ceph/ceph.keyring
sudo ln -sf /var/snap/microceph/current/conf/ceph.conf /etc/ceph/ceph.conf
```

Then retrieve the secret:
```sh
sudo ceph auth get-key client.admin | sudo tee /etc/ceph/admin.secret
sudo chmod 600 /etc/ceph/admin.secret
```

And edit `/etc/fstab` to add the mount point. Edit <NODE_IP_1>,<NODE_IP_2>,<NODE_IP_3> with the IP of each nodes.
```bash
#/etc/fstab
<NODE_IP_1>,<NODE_IP_2>,<NODE_IP_3>:/ /mnt/cephfs ceph name=admin,secretfile=/etc/ceph/admin.secret,fs=docker,_netdev,noatime,x-systemd.automount,x-systemd.mount-timeout=30 0 0
```

To mount the folder, use `sudo mount -a`. Check the mounting point is working my making an empty file in one of the nodes and check on the other nodes if the file exists.

I found out that the folder is not mount on boot everytime (for example if all the nodes went down), so to ensure that Docker have access to the directory, I've added the following configuration file in `/etc/systemd/system/docker.service.d/cephfs-dep.conf`. If the path is not mounted, this file will ask the system to mount it.
```bash
[Unit]
Requires=mnt-cephfs.automount
After=mnt-cephfs.automount

Wants=mnt-cephfs.mount
```

# MariaDB Galera Cluster
Another essential pillar for the project: a redundant database for all websites. MariaDB Galera Cluster is a great solution for this, as it use a "Master-Master" sync. All packages are included with the `mariadb-server` package.

Install the server and rsync, then stop MariaDB from starting:
```sh
sudo apt update
sudo apt install mariadb-server rsync -y
sudo systemctl stop mariadb
```

Create a copy of `/etc/mysql/mariadb.conf.d/60-galera.cnf`, then delete all the content to apply this configuration. Change <NODE_IP> with the current IP of the node, <NODE_NAME> with the hostname of the machine, and <NODE_IP_1>,<NODE_IP_2>,<NODE_IP_3> with the IP of each nodes.
```bash
[galera]
# Activation de Galera
wsrep_on                 = ON
wsrep_provider           = /usr/lib/galera/libgalera_smm.so

# Liste des IP de vos 3 noeuds
wsrep_cluster_address    = "gcomm://<NODE_IP_1>,<NODE_IP_2>,<NODE_IP_3>"
wsrep_cluster_name       = "galera_web_cluster"

# Configuration spécifique au noeud courant
wsrep_node_address       = "<NODE_IP>"
wsrep_node_name          = "<NODE_NAME>"

# Méthode de synchronisation initiale
wsrep_sst_method         = rsync

# Optimisations recommandées pour l'hébergement web (InnoDB)
default_storage_engine   = InnoDB
innodb_autoinc_lock_mode = 2
innodb_force_primary_key = 1
binlog_format            = row
```

Then on the first node (`docker-eva`), bootstrap the cluster:
```sh
sudo galera_new_cluster
sudo systemctl status mariadb
```

If running correctly on the first node, start MariaDB on the other nodes:
```sh
sudo systemctl start mariadb
sudo systemctl enable mariadb
```

On any node, run `sudo mariadb` and check if the cluster is healthy with the following commands:
```mysql
SHOW STATUS LIKE 'wsrep_cluster_size';
SHOW STATUS LIKE 'wsrep_cluster_status';
```

# Keepalived
To reach certain services efficiently, we have to install `keepalived` on each node:
```sh
sudo apt update && sudo apt install -y keepalived
```

Then create the configuration file on each node. Edit `state` to `BACKUP` on the other nodes, same for `priority` (`90` and `80` respectfully) and set a strong secret in `<SECRET>`.
```bash
vrrp_instance VI_1 {
    state MASTER
    interface ens33           
    virtual_router_id 51
    priority 100             
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass <SECRET> 
    }
    virtual_ipaddress {
        10.1.1.10
    }
}
```

Finally, start keepalived with `sudo systemctl enable --now keepalived`

*More step by step info coming soon*
