# -*- mode: ruby -*-
# vi: set ft=ruby :

VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  config.vm.box = "bento/ubuntu-20.10"

  config.vm.provider :virtualbox do |v|
    v.memory = 1024
    v.cpus = 2
    v.linked_clone = true
  end

  # Define three VMs with static private IP addresses.
  boxes = [
    { :name => "worker-1", :ip => "192.168.33.71" },
    { :name => "worker-2", :ip => "192.168.33.72" },
    { :name => "worker-3", :ip => "192.168.33.73" }
  ]

  # Provision each of the VMs.
  boxes.each do |opts|
    config.vm.define opts[:name] do |config|
      config.vm.hostname = opts[:name]
      config.vm.network :private_network, ip: opts[:ip]

      # Provision all the VMs in parallel using Ansible after last VM is up.
      if opts[:name] == "worker-3"
        config.vm.provision "ansible" do |ansible|
          ansible.compatibility_mode = "2.0"
          ansible.playbook = "main.yml"
          ansible.limit = "all"
          ansible.become = true
          ansible.groups = {
            "kubernetes" => ["worker-1", "worker-2", "worker-3"],
            # "kubernetes_master" => [],
            # "kubernetes_master:vars" => {
            #   kubernetes_role: "master",
            #   swapfile_path: "/dev/mapper/ubuntu--vg-ubuntu--lv",
            #   kubernetes_apiserver_advertise_address: "192.168.33.71"
            # },
            "kubernetes_node" => ["worker-1", "worker-2", "worker-3"],
            "kubernetes_node:vars" => {
              kubernetes_role: "node",
              swapfile_path: "/dev/mapper/vagrant--vg-swap_1"
            }
          }
        end
      end
    end
  end

end
