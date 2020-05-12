# -*- mode: ruby -*-
# vim: set ft=ruby :

MACHINES = {
  :mon => {
             :box_name => "centos/7",
             :net => [
                      {ip: '192.168.10.250', adapter: 3, netmask: "255.255.255.0"},
                     ],
             :disk => "NO",
            },
  :sqlnode1 => {
             :box_name => "centos/7",
             :net => [
                      {ip: '192.168.10.21', adapter: 3, netmask: "255.255.255.0"},
                     ],
             :disk => "NO",
            },
  :sqlnode2 => {
             :box_name => "centos/7",
             :net => [
                      {ip: '192.168.10.22', adapter: 3, netmask: "255.255.255.0"},
                     ],
             :disk => "NO",
            },
  :sqlnode3 => {
             :box_name => "centos/7",
             :net => [
                      {ip: '192.168.10.23', adapter: 3, netmask: "255.255.255.0"},
                     ],
             :disk => "NO",
            },
  :backend1 => {
             :box_name => "centos/7",
             :net => [
                      {ip: '192.168.10.10', adapter: 3, netmask: "255.255.255.0"},
                     ],
             :disk => "NO",
            },
  :backend2 => {
             :box_name => "centos/7",
             :net => [
                      {ip: '192.168.10.12', adapter: 3, netmask: "255.255.255.0"},
                     ],
             :disk => "NO",
            },
  :haproxy1 => {
             :box_name => "centos/7",
             :net => [
                      {ip: '192.168.10.100', adapter: 3, netmask: "255.255.255.0"},
                     ],
             :disk => "NO",
            },
  :haproxy2 => {
             :box_name => "centos/7",
             :net => [
                      {ip: '192.168.10.200', adapter: 3, netmask: "255.255.255.0"},
                     ],
             :disk => "NO",
            },
}


hosts_file="127.0.0.1\tlocalhost\n"

MACHINES.each do |hostname,config|  
  config[:net].each do |ip|
      hosts_file=hosts_file+ip[:ip]+"\t"+hostname.to_s+"\n"
  end
end

Vagrant.configure("2") do |config|
  MACHINES.each do |boxname, boxconfig|
    config.vm.define boxname do |box|
        box.vm.box = boxconfig[:box_name]
        box.vm.host_name = boxname.to_s
        boxconfig[:net].each do |ipconf|
          box.vm.network "private_network", ipconf
        end
        box.vm.synced_folder '.', '/vagrant', disabled: true
        box.vm.provider "virtualbox" do |v|
          v.memory = 1024
          v.cpus = 2
          if boxconfig[:disk]=="YES"
            if not File.exists?(boxconfig[:disk_file])
              v.customize ['createhd', '--filename', boxconfig[:disk_file], '--size', 10 * 512]
              v.customize ['storagectl', :id, '--name', 'SATA Controller', '--add', 'sata', '--portcount', 2]
            end
            v.customize ['storageattach', :id,  '--storagectl', 'SATA Controller', '--port', 1, '--device', 0, '--type', 'hdd', '--medium', boxconfig[:disk_file]]
          end
        end
        box.vm.provision "shell" do |shell|
          shell.inline = 'echo -e "$1" > /etc/hosts'
          shell.args = [hosts_file]
        end
        box.vm.provision "ansible" do |ansible|
          ansible.playbook = "playbook.yml"
        end
    end
  end
end

