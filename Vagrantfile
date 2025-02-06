Vagrant.configure("2") do |config|
  config.vm.box = "bento/amazonlinux-2"
  config.vm.network "private_network", ip: "192.168.50.4"
  config.vm.synced_folder "./", "/var/www/html", owner: "vagrant", group: "vagrant"
  config.ssh.private_key_path = "C:/Users/PC_User/.ssh/id_rsa"

end
