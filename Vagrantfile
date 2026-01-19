Vagrant.configure("2") do |config|
  config.vm.box = "bento/ubuntu-22.04"
  config.vm.hostname = "minikube"

  config.vm.network "private_network", ip: "192.168.56.20"

  config.vm.provider "virtualbox" do |vb|
    vb.memory = 4096
    vb.cpus = 2
  end

  # Ansible provisioner
  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "playbook.yaml"
  end
end
