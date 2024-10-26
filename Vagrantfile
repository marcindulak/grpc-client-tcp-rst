# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure(2) do |config|
  config.vm.define "vm1" do |machine|
    machine.vm.box = "ubuntu/jammy64"
    machine.vm.box_url = machine.vm.box
    machine.vm.provider "virtualbox" do |p|
      p.memory = 2048
      p.cpus = 1
    end
  end
  config.vm.define "vm1" do |machine|
    machine.vm.provision :shell, :inline => "hostnamectl set-hostname vm1"
    machine.vm.provision :shell, :inline => "apt-get update && DEBIAN_FRONTEND=noninteractive apt-get install --yes procps vim curl git golang-go iproute2 iptables jq python3 python3-pip python3-venv tcpdump wireshark-common tshark"
  end

  config.vm.define "vm2" do |machine|
    machine.vm.box = "debian/bookworm64"
    machine.vm.box_url = machine.vm.box
    machine.vm.provider "virtualbox" do |p|
      p.memory = 2048
      p.cpus = 1
    end
  end
  config.vm.define "vm2" do |machine|
    machine.vm.provision :shell, :inline => "hostnamectl set-hostname vm2"
    machine.vm.provision :shell, :inline => "apt-get update && DEBIAN_FRONTEND=noninteractive apt-get install --yes procps vim curl git golang-go iproute2 iptables jq python3 python3-pip python3-venv tcpdump wireshark-common tshark"
  end 
end
