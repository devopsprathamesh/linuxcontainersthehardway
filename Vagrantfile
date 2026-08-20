# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/jammy64"
  config.vm.hostname = "containerlab"

  config.vm.network "private_network", type: "dhcp"

  config.vm.provider "virtualbox" do |vb|
    vb.name = "linuxcontainersthehardway"
    vb.cpus = 2
    vb.memory = 2048
  end

  # Only base OS tooling. Everything container-related (overlayfs, namespaces,
  # cgroups, veth, nginx-in-Alpine) is done by hand, following GUIDE.md.
  config.vm.provision "shell", inline: <<-SHELL
    set -euo pipefail
    apt-get update -y
    apt-get install -y iproute2 util-linux iptables curl
  SHELL
end
