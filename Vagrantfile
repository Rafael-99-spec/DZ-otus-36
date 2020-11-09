# -*- mode: ruby -*-
# vim: set ft=ruby :
# -*- mode: ruby -*-
# vim: set ft=ruby :

MACHINES = {

  :server => {
        :box_name => "centos/7",
        :net => [
                   {ip: '192.168.10.10', adapter: 2, netmask: "255.255.255.0", virtualbox__intnet: "vpnnet"},
                ]
  },
  
  #:client => {
  #      :box_name => "centos/7",
  #      :net => [
  #                 {ip: '192.168.10.20', adapter: 2, netmask: "255.255.255.0", virtualbox__intnet: "vpnnet"}
  #              ]
  #},

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

        box.vm.provision "shell", inline: <<-SHELL
          mkdir -p ~root/.ssh
                cp ~vagrant/.ssh/auth* ~root/.ssh
        SHELL
        
        #case boxname.to_s
        #when "server"
        #  box.vm.provision "shell", inline: <<-SHELL
        #     yum install http://www.percona.com/downloads/percona-release/redhat/0.1-6/percona-release-0.1-6.noarch.rpm -y && yum install Percona-Server-server-57 -y
        #    SHELL
        #when "client"
        #  box.vm.provision "shell", path: "client.sh"
        #end

      end

  end
