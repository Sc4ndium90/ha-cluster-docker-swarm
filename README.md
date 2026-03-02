# What's my project?
This repository is a full step by step setup of my upcomming high-availability cluster for my homelab dedicated for hosting Docker containers with Docker Swarm, MicroCeph and MariaDB Galera Cluster. The whole setup also host a Traefik router and Crowdsec for web apps, including monitoring with Grafana and Prometheus.

# Why?
I currently have a 5 nodes Proxmox cluster with no high-availability at all levels, which means that any updates requiring a restart of the node, or any failure of a node implies a loss of availability for any apps hosted on the node with no backups. Also, 3 of the nodes in the Proxmox cluster are not powerfull enough to support recent OSes like Windows 11 or Windows Server when making proof of concepts. This project will make those nodes more useful than before. Also, each time I need a new app, I'll have to make a new virtual machine, installing Docker and add the entry to Traefik by hand. This setup is time consumming and we can loose ourselves in a mess like that (which is also not well documented). Most of the apps hosted are websites or web apps on Docker, with Compose files. There is no need for multiple replicas of a website yet, but this setup will allow this if needed.

# How?
Before breaking all my setup and loose time with my current hardware, I'll virtualize the project in VMWare Workstation 25H2 on my desktop to represent realistically. The nodes I'll use for this project are Dell Wyse 5070 with a 4 core Intel Pentium CPU, 16GB of DDR4 memory and a 256GB SSD. My desktop doesn't have that much RAM and CPU, so I'll stick with 2vCPU, 8GB of memory and 256GB of storage. To represent the homelab network, I'll use a MikroTik CHR router, with the first interface on the NAT network of VMWare, and a LAN Segment network on the second interface.

*Step by step coming soon*
