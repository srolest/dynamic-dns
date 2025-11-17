Vagrant.configure("2") do |config|
  config.vm.box = "debian/bullseye64"

  config.vm.define "dhcp" do |server|
    server.vm.hostname = "dhcp"
    server.vm.network "private_network", virtualbox__intnet: "redinterna1", ip: "192.168.58.30"
  end

  config.vm.define "dns" do |dns|
    dns.vm.hostname = "dns"
    dns.vm.network "private_network", virtualbox__intnet: "redinterna1", ip: "192.168.58.10"
  end

  config.vm.define "c1" do |c1|
  c1.vm.hostname = "c1"
  c1.vm.network "private_network", virtualbox__intnet: "redinterna1", type: "dhcp"
  end
end

# sudo rmmod kvm_amd  # para CPUs AMD
# sudo rmmod kvm
# /var/ib/bind/serafin.test.dns (zona directa)
# /var/ib/bind/serafin.test.rev (zona inversa)