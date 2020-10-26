# vagrant plugin install vagrant-vbguest
# vagrant plugin install vagrant-disksize

Vagrant.configure("2") do |config|
  config.vm.box = "debian/contrib-testing64"
  config.disksize.size = "100GB"

  config.vm.provider "virtualbox" do |vb|
    # vb.gui = true
    vb.memory = "4096"
    vb.cpus = 4
  end

  config.vm.provision "shell", inline: <<-SHELL
    # Prevents prompt during install
    echo '* libraries/restart-without-asking boolean true' | debconf-set-selections
    apt-get update
    apt-get install -y parted build-essential bc curl fakeroot git kernel-wedge quilt ccache flex bison libssl-dev dh-exec \
      crossbuild-essential-armhf crossbuild-essential-arm64 rsync libelf-dev cryptsetup vmdb2 dosfstools qemu-user-static \
      binfmt-support
    echo "fs.inotify.max_user_watches=524288" > /etc/sysctl.d/50-inotify_user_watches.conf
    parted /dev/sda resizepart 1 100%
    resize2fs /dev/sda1
  SHELL
end
