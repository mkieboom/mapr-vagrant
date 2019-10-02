# -*- mode: ruby -*-
# vi: set ft=ruby :

# Install MapR on CentOS using vagrant

ssh_key = "~/.ssh/id_rsa"
box     = "centos/7"

# Add an extra public key for remote accessing the servers
ssh_public_key = "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC8c3uZAc1s+uxfaKdmWbAzxhCRSesUCFKWfpdm0o7R8FKS+VUMxk8xAso+/H9jxsPfC+IO/bDeIYYAtrx/yZ7AKsucI22wvg8WEAZdaZpRbK214HRSwkVpfKNihUFi/JE0BiCScjkF1DPmfiApYZLTelJyoU68AJgaWG0i6khq+YwXI2ON5SXpPblvIASqD20LljTLjcus69ZhzoQAgWJ8ixE/eLDXnxwwqwUK8gMnCNzYblemyZ6roV4e24qjw9IE7lpc47yO3MuKoTtVMqwqzdAn1W3yMQIReChEBJYRTaJsQUFQjBr+jELkbSNhJv8nJn7OOO9yd/+jygYZ+NtX"


file_to_disk = "/tmp/maprn02_sdb.vdi"
#file_to_disk = "mydisk.vmdk"

# List of servers
# NOTE: in reverse order to run ansible from the last node in the servers list
servers = [
  { :hostname => "maprn02", :hostonly_ip => "192.168.168.16", :bridged_ip => "192.168.1.16", :bridged_adapter=> "eno1", :ram => 6144, :cpu => 2, :ansible_group => "all" }
]


Vagrant.configure(2) do |config|
  if Vagrant.has_plugin?("vagrant-hostmanager")
    config.hostmanager.enabled = true
    config.hostmanager.manage_host = true
    config.hostmanager.manage_guest = true
    config.hostmanager.ignore_private_ip = false
    config.hostmanager.include_offline = false
  end #end if


  # Configure the servers
  servers.each_with_index do |server, index|
      
    # Create hosts file on all the servers
    config.vm.provision :shell, inline: "sudo echo #{server[:hostonly_ip]} #{server[:hostname]} | sudo tee -a /etc/hosts"

    box_image = server[:box] ? server[:box] : box;
    config.vm.define server[:hostname] do |conf|

      conf.vm.box = box_image.to_s
      conf.vm.hostname = server[:hostname]

      # Host only network config
      net_config_hostonly = {}
      if server[:hostonly_ip] != "dhcp"
        net_config_hostonly[:ip] = server[:hostonly_ip]
        net_config_hostonly[:netmask] = server[:netmask] || "255.255.255.0"
      else
        net_config_hostonly[:type] = "dhcp"
      end
      conf.vm.network "private_network", net_config_hostonly

      # Bridged network config
      net_config_bridged = {}
      net_config_bridged[:bridge] = server[:bridged_adapter]
      if server[:bridged_ip] != "dhcp"
        net_config_bridged[:ip] = server[:bridged_ip]
        net_config_bridged[:netmask] = server[:netmask] || "255.255.255.0"
      else
        net_config_bridged[:type] = "dhcp"
      end
      conf.vm.network "public_network", net_config_bridged

      # Configure port forwarding (optional)
      if !server[:port_guest].nil? && !server[:port_host].nil?
        conf.vm.network "forwarded_port", guest: server[:port_guest], host: server[:port_host]
      end

      # Configure shared folders (optional)
      if !server[:folder_guest].nil? && !server[:folder_host].nil?
        conf.vm.synced_folder server[:folder_host], server[:folder_guest]
      end

      # Confogure the machine CPU and memory
      cpu = server[:cpu] ? server[:cpu] : 1;
      memory = server[:ram] ? server[:ram] : 512;
      config.vm.provider "virtualbox" do |vb|
        vb.customize ["modifyvm", :id, "--cpus", cpu.to_s]
        vb.customize ["modifyvm", :id, "--memory", memory.to_s]
        vb.name = server[:hostname] + "_" + server[:bridged_ip]

        unless File.exist?(file_to_disk)
          #vb.customize [ "createmedium", "disk", "--filename", "mydisk.vmdk", "--format", "vmdk", "--size", 1024 * 500 ]
          vb.customize ['createhd', '--filename', file_to_disk, '--size', 500 * 1024]
        end
        vb.customize [ "storageattach", server[:hostname] + "_" + server[:bridged_ip] , "--storagectl", "IDE", "--port", "1", "--device", "0", "--type", "hdd", "--medium", file_to_disk]
      end


      # Add the private and public key for keyless ssh between the servers
      config.ssh.private_key_path = ["~/.vagrant.d/insecure_private_key", ssh_key]
      config.ssh.insert_key = false
      config.vm.provision "file", source: ssh_key , destination: "~/.ssh/id_rsa"
      config.vm.provision "file", source: ssh_key + ".pub", destination: "~/.ssh/authorized_keys"

      # Also add my personal public key to the authorized keys file
      config.vm.provision :shell, inline: "echo #{ssh_public_key} >> ~/.ssh/authorized_keys", privileged: false

      # Add the private and public key for the root user as well to run ansible as root
      config.vm.provision "shell", inline: <<-EOF
        sudo mkdir -p /root/.ssh/
        sudo cp /home/vagrant/.ssh/id_rsa /root/.ssh/id_rsa
        sudo cat /home/vagrant/.ssh/authorized_keys >> /root/.ssh/authorized_keys
      EOF

      # Remove 127.0.0.1 <hostname> from /etc/hosts to allow correct IP lookup
      config.vm.provision :shell, inline: "sed -i'' '/^127.0.0.1\\t#{conf.vm.hostname}\\t#{conf.vm.hostname}$/d' /etc/hosts"

      # Install some basic tools
      if index == servers.size - 1
        config.vm.provision "shell", inline: <<-EOF
          yum install -y epel-release
          yum install -y net-tools wget

          # Install a single node MapR cluster
          curl https://raw.githubusercontent.com/mkieboom/mapr-ansible/master/vagrant/1node_unsecure_minimum.sh | bash

        EOF
      end #end if
    end #end conf    
  end #end servers
end #end config
