# -*- mode: ruby -*-
# vim: set ft=ruby :

MACHINES = {
  :zabbix => {
        :box_name => "centos/8",
        :box_version => "2011.0",
        :ip_addr => '192.168.11.101',
  },
}

Vagrant.configure("2") do |config|
config.vm.synced_folder ".", "/vagrant", disabled: true
  MACHINES.each do |boxname, boxconfig|

      config.vm.define boxname do |box|

          box.vm.box = boxconfig[:box_name]
          box.vm.host_name = boxname.to_s

          box.vm.host_name = "zabbix"

          box.vm.network "private_network", ip: boxconfig[:ip_addr]

          box.vm.provider :virtualbox do |vb|
            	  vb.customize ["modifyvm", :id, "--memory", "2048"]
		  end
 	  box.vm.provision "shell", inline: <<-SHELL
          SHELL
      end
  end
end

