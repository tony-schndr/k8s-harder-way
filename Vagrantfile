# -*- mode: ruby -*-
# vi: set ft=ruby :

VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  config.vm.box = "bento/ubuntu-20.04"

  config.vm.provider :virtualbox do |v|
    v.memory = 1024
    v.cpus = 2
    v.linked_clone = true
  end

  # Define VMs with static private IP addresses.
  boxes = [
    { :name => "etcd-1", :ip => "10.240.0.11" },
    { :name => "etcd-2", :ip => "10.240.0.12" },
    { :name => "etcd-3", :ip => "10.240.0.13" },
    { :name => "controller-1", :ip => "10.240.0.21" },
    { :name => "controller-2", :ip => "10.240.0.22" },
    { :name => "worker-1", :ip => "10.240.0.31" },
    { :name => "worker-2", :ip => "10.240.0.32"}
  ]

  # Provision each of the VMs.
  boxes.each do |opts|
    config.vm.define opts[:name] do |config|
      config.vm.hostname = opts[:name]
      config.vm.network :private_network, ip: opts[:ip]
      if opts[:name] == "worker-1" || opts[:name] == "worker-2"
        config.vm.provider :virtualbox do |vb|
          vb.customize ["modifyvm", :id, "--memory", "2048"]
          vb.customize ["modifyvm", :id, "--cpus", "2"]
        end
      end
      if opts[:name] == "controller-1" || opts[:name] == "controller-2"
        config.vm.provider :virtualbox do |vb|
          vb.customize ["modifyvm", :id, "--memory", "512"]
          vb.customize ["modifyvm", :id, "--cpus", "2"]
        end
      end
      if opts[:name] == "etcd-1" || opts[:name] == "etcd-2" || opts[:name] == "etcd-3"
        config.vm.provider :virtualbox do |vb|
          vb.customize ["modifyvm", :id, "--memory", "512"]
          vb.customize ["modifyvm", :id, "--cpus", "2"]
        end
      end
      # Provision all the VMs in parallel using Ansible after last VM is up.
      if opts[:name] == "worker-2"
        config.vm.provision "ansible" do |ansible|
          ansible.compatibility_mode = "2.0"
          ansible.playbook = "main.yml"
          ansible.limit = "all"
          ansible.become = true
          ansible.groups = {
            "kubernetes" => ["worker-1", "worker-2",
                             "controller-1", "controller-2", "etcd-1", "etcd-2", "etcd-3"],
            "controller" => ["controller-1", "controller-2"],
            "etcd" => ["etcd-1", "etcd-2", "etcd-3"],
            # "kubernetes_master:vars" => {
            #   kubernetes_role: "master",
            #   swapfile_path: "/dev/mapper/ubuntu--vg-ubuntu--lv",
            #   kubernetes_apiserver_advertise_address: "192.168.33.71"
            # },
            "worker" => ["worker-1", "worker-2"],
            "worker:vars" => {
              kubernetes_role: "worker",
              #swapfile_path: "/dev/mapper/vagrant--vg-swap_1"
            }
          }
        end
      end
    end
  end

end
