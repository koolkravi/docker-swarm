
ENV['VAGRANT_NO_PARALLEL']='yes'
Vagrant.configure("2") do |config|
  box_name="hashicorp/bionic64"
  config.vm.box = "base"
  nodes =2
  (1..nodes).each do |i|
      config.vm.define "ubuntuvm0#{i}" do |node|
          node.vm.box=box_name
          node.vm.network "private_network", ip: "10.0.0.#{i+4}"
          node.vm.hostname="ubuntuvm0#{i}"
          node.vm.provider "virtualbox" do |vb|
            vb.name="ubuntuvm0#{i}"
            vb.memory =1024
            vb.cpus=1
          end
       end
   end
end
