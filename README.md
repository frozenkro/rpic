rpic
====

Ansible playbook to provision a 3-node Raspberry Pi k3s cluster.

Nodes:
  rpic1 (10.0.0.1)  k3s server
  rpic3 (10.0.0.2)  k3s agent
  rpic4 (10.0.0.3)  k3s agent

WiFi is used for Ansible provisioning and internet.
Ethernet switch handles k3s cluster traffic.

Run:
  ansible-playbook -i inventory.yml provision.yml --ask-pass -K

What it does:
  - enables cgroup_memory in cmdline.txt
  - assigns static IPs on eth0 via NetworkManager
  - installs and auth's Tailscale
  - installs k3s (server on rpic1, agents join via eth0)
  - reboots to apply cmdline changes

Config:
  inventory.yml          hosts and per-host vars (ip, role)
  group_vars/rpic.yml    tailscale auth key, k3s url/token
  provision.yml          the playbook
  user_data.sh           my original shell attempt (superseded)
