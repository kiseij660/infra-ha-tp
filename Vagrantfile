Vagrant.configure("2") do |config|

  config.vm.provider "vmware_desktop" do |v|
    v.gui = false
  end

  config.vm.define "admin" do |admin|
    admin.vm.box = "bento/ubuntu-22.04"
    admin.vm.hostname = "admin"
    admin.vm.network "private_network", ip: "192.168.100.10"
  end

  config.vm.define "node01" do |node|
    node.vm.box = "bento/rockylinux-9"
    node.vm.hostname = "node01"
    node.vm.network "private_network", ip: "192.168.100.11"
  end

  config.vm.define "node02" do |node|
    node.vm.box = "bento/rockylinux-9"
    node.vm.hostname = "node02"
    node.vm.network "private_network", ip: "192.168.100.12"
  end

end
