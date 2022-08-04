# export ENV variables (source .env file)
rhsm_username     = ENV["RH_SUBSCRIPTION_MANAGER_USERNAME"]
rhsm_password     = ENV["RH_SUBSCRIPTION_MANAGER_PASSWORD"]

# Check if the required environment variables are set
if !rhsm_username or !rhsm_password
  puts "Required environment variables not found. Please set RH_SUBSCRIPTION_MANAGER_USERNAME and RH_SUBSCRIPTION_MANAGER_PASSWORD"
  abort
end

# RedHat Subscription Manager - Register Script (developer)
rhsm_register_script = <<-SCRIPT
if ! subscription-manager status; then
  syspurpose set-role "Red Hat Enterprise Linux Server"
  syspurpose set-sla "Self-Support"
  syspurpose set-usage "Development/Test"
  sudo subscription-manager register \
    --username=#{rhsm_username} \
    --password=#{rhsm_password} \
    --auto-attach
fi
syspurpose show
sudo subscription-manager status
SCRIPT

# RedHat Subscription Manager - Unregister Script
rhsm_unregister_script = <<-SCRIPT
if subscription-manager status; then
  sudo subscription-manager unregister
fi
SCRIPT

VAGRANTFILE_API_VERSION = "2"
VAGRANT_DISABLE_VBOXSYMLINKCREATE = "1"

file_to_disk1 = './disk-0-1.vdi'
file_to_disk2 = './disk-0-2.vdi'

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|

  # Use same SSH key for each machine
  config.ssh.insert_key = false
  config.vm.box_check_update = false

  # Server 2 Configuration
  config.vm.define "server2" do |server2|
    server2.vm.box = "rdbreak/rhel8node"
    # server2.vm.hostname = "server2.eight.example.com"
    server2.vm.network "private_network", ip: "192.168.56.151"
    server2.vm.network "private_network", ip: "192.168.56.175"
    server2.vm.network "private_network", ip: "192.168.56.176"
    server2.vm.synced_folder ".", "/vagrant", type: "rsync", rsync__exclude: ".git/", rsync__exclude: "*.vdi"
    server2.vm.provider "virtualbox" do |server2|

      unless File.exist?(file_to_disk1)
        server2.customize ['storagectl', :id, '--name', 'SATA Controller', '--add', 'sata', '--portcount', 2]
      end

      unless File.exist?(file_to_disk1)
        server2.customize ['createhd', '--filename', file_to_disk1, '--variant', 'Fixed', '--size', 16 * 1024]
      end

      unless File.exist?(file_to_disk2)
        server2.customize ['createhd', '--filename', file_to_disk2, '--variant', 'Fixed', '--size', 16 * 1024]
      end

      server2.memory = "2048"
      server2.customize ['storageattach', :id,  '--storagectl', 'SATA Controller', '--port', 1, '--device', 0, '--type', 'hdd', '--medium', file_to_disk1]
      server2.customize ['storageattach', :id,  '--storagectl', 'SATA Controller', '--port', 2, '--device', 0, '--type', 'hdd', '--medium', file_to_disk2]

    end

    server2.vm.provision "shell", inline: <<-SHELL
      if mount | grep /dev/sdb > /dev/null; then
        echo "/dev/sdb is already mounted"
      else
        yes|  mkfs.ext4 -L extradisk1 /dev/sdb
      fi
    SHELL
    server2.vm.provision "shell", inline: <<-SHELL
      if cat /etc/fstab | grep /extradisk1 > /dev/null; then
        echo "/extradisk1 already exists in /etc/fstab"
      else 
        mkdir /extradisk1 ; echo \'LABEL=extradisk1 /extradisk1 ext4 defaults 0 0\' >> /etc/fstab
      fi
    SHELL
    server2.vm.provision "shell", inline: <<-SHELL
      if mount | grep /dev/sdc > /dev/null; then
        echo "/dev/sdc is already mounted"
      else
        yes|  mkfs.ext4 -L extradisk2 /dev/sdc
      fi
    SHELL
    server2.vm.provision "shell", inline: <<-SHELL
      if cat /etc/fstab | grep /extradisk1 > /dev/null; then
        echo "/extradisk2 already exists in /etc/fstab"
      else 
        mkdir /extradisk2 ; echo \'LABEL=extradisk2 /extradisk2 ext4 defaults 0 0\' >> /etc/fstab
      fi
    SHELL

    server2.vm.provision :ansible_local do |ansible|
      ansible.playbook = "/vagrant/playbooks/server2.yml"
      ansible.install = false
      ansible.compatibility_mode = "2.0"
      ansible.inventory_path = "/vagrant/inventory"
      ansible.config_file = "/vagrant/ansible.cfg"
      ansible.limit = "all"
    end

    # Run RedHat Subscription Register Script
    server2.vm.provision :shell, inline: rhsm_register_script

    # Run RedHat Subscription Unregister Script
    server2.trigger.before :destroy do |trigger|
      trigger.name = "Before Destroy trigger"
      trigger.info = "Unregistering this VM from RedHat Subscription Manager..."
      trigger.warn = "If this fails, unregister VMs manually at https://access.redhat.com/management/subscriptions"
      trigger.run_remote = {inline: rhsm_unregister_script}
      trigger.on_error = :continue
    end

    server2.vm.provision :shell, :inline => "sudo update -y", run: "always"

    server2.vm.provision :shell, :inline => "reboot", run: "always"

  end

  # Repo Configuration
  config.vm.define "repo" do |repo|
    repo.vm.box = "rdbreak/rhel8repo"
    # repo.vm.hostname = "repo.example.com"
    repo.vm.provision :shell, :inline => "sudo rm -f /EMPTY;", run: "always"
    repo.vm.provision :shell, :inline => "sudo sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config; sudo systemctl restart sshd;", run: "always"
    repo.vm.provision :shell, :inline => "yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm -y; sudo yum install -y sshpass python3-pip python3-devel httpd vsftpd createrepo", run: "always"
    repo.vm.provision :shell, :inline => " python3 -m pip install -U pip ; python3 -m pip install pexpect; python3 -m pip install ansible", run: "always"
    repo.vm.synced_folder ".", "/vagrant", type: "rsync", rsync__exclude: ".git/", rsync__exclude: "*.vdi"
    repo.vm.network "private_network", ip: "192.168.56.149"

    repo.vm.provider "virtualbox" do |repo|
      repo.memory = "1024"
    end
  end

  # Server 1 Configuration
  config.vm.define "server1" do |server1|
    server1.vm.box = "rdbreak/rhel8node"
    server1.vm.synced_folder ".", "/vagrant", type: "rsync", rsync__exclude: ".git/", rsync__exclude: "*.vdi"
    # server1.vm.hostname = "server1.eight.example.com"
    server1.vm.network "private_network", ip: "192.168.56.150"
    server1.vm.provider :virtualbox do |server1|
      server1.customize ['modifyvm', :id,'--memory', '2048']
    end

    server1.vm.provision :ansible_local do |ansible|
      ansible.playbook = "/vagrant/playbooks/master.yml"
      ansible.install = false
      ansible.compatibility_mode = "2.0"
      ansible.inventory_path = "/vagrant/inventory"
      ansible.config_file = "/vagrant/ansible.cfg"
      ansible.limit = "all"
    end

    # Run RedHat Subscription Register Script
    server1.vm.provision :shell, inline: rhsm_register_script

    # Run RedHat Subscription Unregister Script
    server1.trigger.before :destroy do |trigger|
      trigger.name = "Before Destroy trigger"
      trigger.info = "Unregistering this VM from RedHat Subscription Manager..."
      trigger.warn = "If this fails, unregister VMs manually at https://access.redhat.com/management/subscriptions"
      trigger.run_remote = {inline: rhsm_unregister_script}
      trigger.on_error = :continue
    end

    server1.vm.provision :shell, :inline => "sudo update -y", run: "always"

    server1.vm.provision :shell, :inline => "reboot", run: "always"

  end
end



