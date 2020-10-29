# vagrant plugin install vagrant-vbguest
# vagrant plugin install vagrant-disksize

Vagrant.configure("2") do |config|
  config.vm.box = "debian/contrib-testing64"
  config.disksize.size = "100GB"

  config.ssh.forward_agent = true

  config.vm.provider "virtualbox" do |vb|
    # vb.gui = true
    vb.memory = "4096"
    vb.cpus = 4
  end

  config.vm.provision "shell", inline: <<-SHELL
    set -euo pipefail

    # Make sure the FS is resized too
    # yes | parted /dev/sda resizepart 1 100% # doesn't seem to be needed
    resize2fs /dev/sda1

    # Setup kernel repo
    wget -qO - https://lstoll-kernels-apt.s3.amazonaws.com/GPGKEY | gpg --dearmor - > /etc/apt/trusted.gpg.d/B26F07E7.gpg
    echo "deb http://lstoll-kernels-apt.s3.amazonaws.com buster main" > /etc/apt/sources.list.d/lstoll-kernels.list

    # Prevents prompt during install
    DEBIAN_FRONTEND=noninteractive apt-get update
    DEBIAN_FRONTEND=noninteractive apt-get install -y parted build-essential bc \
      curl fakeroot git kernel-wedge quilt ccache flex bison libssl-dev dh-exec \
      crossbuild-essential-armhf crossbuild-essential-arm64 rsync libelf-dev \
      cryptsetup vmdb2 dosfstools qemu-user-static \
      binfmt-support time linux-image-5.9.0-1-lstoll-amd64-unsigned

    # vscode needs this for watches
    echo "fs.inotify.max_user_watches=524288" > /etc/sysctl.d/50-inotify_user_watches.conf

    # git remote set-url origin git@github.com:lstoll/debian-arm-images.git
    test -d /home/vagrant/debian-arm-images || (cd /home/vagrant && git clone https://github.com/lstoll/debian-arm-images && chown -R vagrant:vagrant debian-arm-images)
    # git remote set-url origin git@github.com:lstoll/debian-kernel.git
    test -d /home/vagrant/debian-kernel || (cd /home/vagrant && git clone https://github.com/lstoll/debian-kernel && chown -R vagrant:vagrant debian-kernel && cd debian-kernel && git remote add upstream https://salsa.debian.org/kernel-team/linux.git)
    # git remote set-url origin git@github.com:lstoll/vmdb2.git
    test -d /home/vagrant/vmdb2 || (cd /home/vagrant && git clone https://github.com/lstoll/vmdb2 && chown -R vagrant:vagrant vmdb2 && cd vmdb2 && git checkout lstoll && git remote add upstream git://git.liw.fi/vmdb2)
  SHELL
end
