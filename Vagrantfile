# -*- mode: ruby -*-
# vi: set ft=ruby :

VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  config.vm.box = "generic/ubuntu2010"
  config.vm.provider :libvirt do |libvirt|
    libvirt.qemu_use_session = false
  end
  # Define VMs with static private IP addresses.
                            
                            
                            
  boxes = [
    { :name => "etcd-1", :ip => "192.168.1.11"}, 
    { :name => "etcd-2", :ip => "192.168.1.12"},
    { :name => "etcd-3", :ip => "192.168.1.13"},
    # { :name => "controller", :ip => "192.168.1.20" },
    { :name => "controller-1", :ip => "192.168.1.21"},
    { :name => "controller-2", :ip => "192.168.1.22"},
    { :name => "worker-1", :ip => "192.168.1.31",},
    { :name => "worker-2", :ip => "192.168.1.32",}
  ]

  # Provision each of the VMs.
  boxes.each do |opts|
    config.vm.define opts[:name] do |config|
      config.vm.hostname = opts[:name]
      config.vm.network :private_network, ip: opts[:ip], :libvirt__netmask => '255.255.255.0',:libvirt__network_name => 'k8shwnetwork'
      if opts[:name] == "worker-1" || opts[:name] == "worker-2"
        config.vm.provider :libvirt do |libvirt|
          libvirt.cpus = 2
          libvirt.memory = 2048
        end
      end
      if opts[:name] == "controller-1" || opts[:name] == "controller-2"
        config.vm.provider :libvirt do |libvirt|
          libvirt.cpus = 2
          libvirt.memory = 2048
        end
      end
      if opts[:name] == "etcd-1" || opts[:name] == "etcd-2" || opts[:name] == "etcd-3"
        config.vm.provider :libvirt do |libvirt|
          libvirt.cpus = 2
          libvirt.memory = 2048
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
