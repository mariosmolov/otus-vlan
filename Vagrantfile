# -*- mode: ruby -*-
# vim: set ft=ruby :

MACHINES = {
  :inetRouter => {
        :box_name => "centos/7",
        :net => [
                   {adapter: 2, auto_config: false, virtualbox__intnet: "team-net"},
                   {adapter: 3, auto_config: false, virtualbox__intnet: "team-net"}
                ]
  },

  :centralRouter => {
        :box_name => "centos/7",
        :net => [
                   {adapter: 2, auto_config: false, virtualbox__intnet: "team-net"},
                   {adapter: 3, auto_config: false,virtualbox__intnet: "team-net"},
                   {ip: '192.168.255.5',adapter: 4, netmask: "255.255.255.252", virtualbox__intnet: "central-net"}
                ]
  },

  :officeRouter => {
        :box_name => "centos/7",
        :net => [
               {adapter: 2, auto_config: false, virtualbox__intnet: "central-net"},
               {adapter: 3, auto_config: false, virtualbox__intnet: "test-net"}
            ]
  },

  :testServer1 => {
    :box_name => "centos/7",
    :net => [
               {adapter: 2, auto_config: false, virtualbox__intnet: "test-net"},
            ]
  },

  :testClient1 => {
    :box_name => "centos/7",
    :net => [
               {adapter: 2, auto_config: false, virtualbox__intnet: "test-net"},
            ]
  },

  :testServer2 => {
    :box_name => "centos/7",
    :net => [
               {adapter: 2, auto_config: false, virtualbox__intnet: "test-net"},
            ]
  },

  :testClient2 => {
    :box_name => "centos/7",
    :net => [
               {adapter: 2, auto_config: false, virtualbox__intnet: "test-net"},
            ]
  }

}

Vagrant.configure("2") do |config|

  MACHINES.each do |boxname, boxconfig|

    config.vm.define boxname do |box|

        box.vm.box = boxconfig[:box_name]
        box.vm.host_name = boxname.to_s

        boxconfig[:net].each do |ipconf|
          box.vm.network "private_network", ipconf
        end

        if boxconfig.key?(:public)
          box.vm.network "public_network", boxconfig[:public]
        end

	box.vm.provider :virtualbox do |vb|
            vb.customize ["modifyvm", :id, "--memory", "256"]
        end

        box.vm.provision "shell", inline: <<-SHELL
          mkdir -p ~root/.ssh
          cp ~vagrant/.ssh/auth* ~root/.ssh
        SHELL

        box.vm.provision "ansible" do |ansible|
	    ansible.compatibility_mode = "2.0"
	    ansible.become = "true"
	    ansible.playbook = "provisioning/playbook.yml"
        end

    end

  end

end

