# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  # Define the three Rocky Linux boxes
  (1..3).each do |i|
    config.vm.define "vault#{i}" do |node|
      # Use Rocky Linux 9 box from Vagrant Cloud
      node.vm.box = "rockylinux/9"
	  node.vm.box_version = "4.0.0"
	  node.vm.network "private_network", ip: "192.168.60.#{20 + i}"
      
      # Configure SSH port forwarding
      node.vm.network "forwarded_port", guest: 22, host: 20021 + i, id: "ssh"
      
      # VirtualBox provider settings
      node.vm.provider "virtualbox" do |vb|
        vb.memory = "1024"
        vb.cpus = 1
      end
      
      # Ensure SSH is enabled
      node.vm.provision "shell", inline: <<-SHELL
        sudo systemctl enable sshd
        sudo systemctl start sshd
        sudo dnf install -y podman systemd dbus python3-pip git wget curl
        pip3 install podman-compose
        sudo -u vagrant podman system migrate || true
        podman run hello-world
		# it aint running
		#sudo firewall-cmd --permanent --add-port=8200/tcp
    	#sudo firewall-cmd --permanent --add-port=8201/tcp
    	#sudo firewall-cmd --reload

		echo "192.168.60.21 vault1" >> /etc/hosts
        echo "192.168.60.22 vault2" >> /etc/hosts
        echo "192.168.60.23 vault3" >> /etc/hosts

		sudo hostnamectl set-hostname vault#{i}
      SHELL
    end
  end
end