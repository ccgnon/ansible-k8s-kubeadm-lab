# -*- mode: ruby -*-
# Labo Kubernetes : 1 control-plane + 2 workers (EL9)
#
# Box : AlmaLinux 9 par défaut (binaire-compatible RHEL, dépôts accessibles
# sans abonnement). Pour une vraie RHEL : BOX=generic/rhel9 vagrant up
# puis enregistrer les VM (subscription-manager register).
BOX = ENV.fetch("BOX", "almalinux/9")

NODES = [
  { name: "redhat-vm1", ip: "192.168.56.11", cpus: 2, memory: 2048 },  # control-plane : 2 vCPU / 2 Go minimum
  { name: "redhat-vm2", ip: "192.168.56.12", cpus: 2, memory: 2048 },
  { name: "redhat-vm3", ip: "192.168.56.13", cpus: 2, memory: 2048 },
]

Vagrant.configure("2") do |config|
  config.vm.box = BOX
  config.vm.box_check_update = false
  # Clé SSH propre à chaque VM (utilisée par l'inventaire Ansible)
  config.ssh.insert_key = true
  config.vm.synced_folder ".", "/vagrant", disabled: true

  NODES.each do |node|
    config.vm.define node[:name] do |vm|
      vm.vm.hostname = node[:name]
      vm.vm.network "private_network", ip: node[:ip]
      vm.vm.provider "virtualbox" do |vb|
        vb.name   = node[:name]
        vb.cpus   = node[:cpus]
        vb.memory = node[:memory]
        vb.linked_clone = true
      end
    end
  end
end
