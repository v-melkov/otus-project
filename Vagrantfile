# -*- mode: ruby -*-
# vim: set ft=ruby :
home = ENV['HOME']
ENV["LC_ALL"] = "en_US.UTF-8"

MACHINES = {

  firewall: {
    box_name: 'centos/7',
    net: [
      { ip: '192.168.110.100', adapter:2, netmask: '255.255.255.0' },
      { ip: '192.168.255.1', adapter: 3, netmask: '255.255.255.252', virtualbox__intnet: 'dmz-net' },
    ]
  },
  frontend: {
    box_name: "centos/7",
    net: [
      { ip: '192.168.110.105', netmask: '255.255.255.0' }, # для настройки ансиблом
      { ip: '192.168.255.2', netmask: '255.255.255.252', virtualbox__intnet: 'dmz-net' },
    ]
  },
  apache: {
    box_name: "centos/7",
    net: [
      { ip: '192.168.110.102', netmask: '255.255.255.0' },
    ]
  },
  mysql: {
    box_name: "centos/7",
    net: [
      { ip: '192.168.110.136', netmask: '255.255.255.0' },
    ]
  },
  log: {
    box_name: "centos/7",
    net: [
      { ip: '192.168.110.103', netmask: '255.255.255.0' },
    ]
  },
  backup: {
    box_name: "centos/7",
    net: [
      { ip: '192.168.110.104', netmask: '255.255.255.0' },
    ]
  },
}

Vagrant.configure("2") do |config|
  config.vbguest.no_install = true
  config.vm.synced_folder '.', '/vagrant', disabled: true
  MACHINES.each do |boxname, boxconfig|
    config.vm.define boxname do |box|
      box.vm.box = boxconfig[:box_name]
      box.vm.host_name = boxname.to_s
      boxconfig[:net].each do |ipconf|
        box.vm.network 'private_network', ipconf
      end
      box.vm.network 'public_network', boxconfig[:public] if boxconfig.key?(:public)
      box.vm.provider :virtualbox do |vb|
        vb.customize ["modifyvm", :id, "--memory", "1024"]
      end

    end
  end
  # для тестирования сразу включим форвардинг
  # в ансибле есть задача включения форвардинга
  config.vm.define 'firewall' do |firewall|
    firewall.vm.provision 'shell', run: 'always', inline: <<-SHELL
      echo "net.ipv4.conf.all.forwarding=1" >> /etc/sysctl.d/01-forwarding.conf
      echo "net.ipv4.ip_forward=1" >> /etc/sysctl.d/01-forwarding.conf
      iptables -t nat -A POSTROUTING ! -d 192.168.0.0/16 -o eth0 -j MASQUERADE
      sysctl -p /etc/sysctl.d/01-forwarding.conf
    SHELL
  end
  config.vm.define 'backup' do |backup|
    backup.vm.provision :ansible do |ansible|
      ansible.limit = "all"
      ansible.playbook = "playbook.yml"
      ansible.inventory_path = "hosts"
    end
  end
end
