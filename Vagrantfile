# -*- mode: ruby -*-
# vi: set ft=ruby :

VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  config.vm.box = "generic/ubuntu2004"
  config.vm.provider :libvirt do |libvirt|
    libvirt.qemu_use_session = false
  end
  boxes = [
    { :name => "controller-1", :ip => "192.168.1.21"},
    { :name => "controller-2", :ip => "192.168.1.22"},
    { :name => "controller-3", :ip => "192.168.1.23"},
    { :name => "worker-1", :ip => "192.168.1.31",},
    { :name => "worker-2", :ip => "192.168.1.32",}
  ]

  boxes.each do |opts|
    config.vm.define opts[:name] do |config|
      config.vm.hostname = opts[:name]
      config.vm.network :private_network, ip: opts[:ip], :libvirt__netmask => '255.255.255.0', :libvirt__network_name => 'k8shwnetwork'
      if opts[:name] == "worker-1" || opts[:name] == "worker-2"
        config.vm.provider :libvirt do |libvirt|
          libvirt.cpus = 2
          libvirt.memory = 2048
        end
      end
      if opts[:name] == "controller-1" || opts[:name] == "controller-2" || opts[:name] == "controller-3"
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
                             "controller-1", "controller-2","controller-3", "etcd-1", "etcd-2", "etcd-3"],
            "controller" => ["controller-1", "controller-2", "controller-3"],
            "etcd" => ["controller-1", "controller-2", "controller-3"],
            "worker" => ["worker-1", "worker-2"],
            "worker:vars" => {
              kubernetes_role: "worker",
            }
          }
        end
      end
    end
  end

end
