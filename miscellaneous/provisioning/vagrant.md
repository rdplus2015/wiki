# Vagrant

## Introduction

Vagrant is a tool for managing virtualized development environments. It simplifies setting up, maintaining, and distributing development environments.

### Important Concepts

- **Vagrant Box**: A prepackaged environment that contains an operating system and necessary tools. It serves as the base image for Vagrant environments.
- **Vagrantfile**: The configuration file written in Ruby that defines the settings and behavior of a Vagrant environment.

### Ressources

- [Installation ](https://developer.hashicorp.com/vagrant/docs/installation)
- [Documentation](https://developer.hashicorp.com/vagrant)
- [Vagrant Cloud](https://portal.cloud.hashicorp.com/vagrant/discover)

## Basic Commands

commands can take options and arguments `Usage: vagrant [options] <command> [<args>]`

```sh
vagrant init <box-name>       # Initialize Vagrantfile
vagrant up                    # Start VM
vagrant halt                  # Stop VM
vagrant reload                # Restart VM
vagrant destroy               # Delete VM
vagrant status                # Check VM status
vagrant ssh                   # SSH into VM
vagrant suspend               # Pause VM
vagrant resume                # Resume VM
vagrant global-status         # List all running VMs
vagrant global-status --prune # List all VMs and removes outdated entries for VMs that no longer exist.

```

## Box Management

```sh
vagrant box add <box-name>     # Add a new base box
vagrant box list               # List available boxes
vagrant box remove <box-name>  # Remove a box
vagrant box outdated           # Check for updates
vagrant box update             # Update all boxes
```

## Vagrantfile Configuration

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/bionic64"
  config.vm.network "private_network", type: "dhcp"
  config.vm.synced_folder "./data", "/vagrant_data"
  config.vm.provider "virtualbox" do |vb|
    vb.memory = "1024"
    vb.cpus = 2
  end
end
```

## Provisioning

### Shell Provisioning

```ruby
config.vm.provision "shell", inline: <<-SHELL
  sudo apt-get update
  sudo apt-get install -y apache2
SHELL
```

### Ansible Provisioning

```ruby
config.vm.provision "ansible" do |ansible|
  ansible.playbook = "playbook.yml"
end
```

### Puppet Provisioning

```ruby
config.vm.provision "puppet" do |puppet|
  puppet.manifest_file = "site.pp"
end
```

```sh
vagrant reload --provision  # Reload provisioning
```

## Networking

### 1. **Cconfig.vm.network "private_network", type: "dhcp"**

Create an **internal** network between your PC and the VM.

- Automatically assigned IP address (DHCP).
- Accessible only from your host.
- Useful for local development (e.g. access to an app via ˀhttp://192.168.x.x`).

```ruby
config.vm.network "private_network", type: "dhcp"
```

### 2. **Cconfig.vm.network "private_network", ip: "192.168.33.10"**

Same type as the previous one, but **Fixed IP**.

- You set the address manually.
- Practical if you want to access your VM always via the same IP.

```ruby
config.vm.network "private_network", ip: "192.168.33.10"
```

### 3. **Cconfig.vm.network "public_network"**

Connect your VM directly to your host's **physical network** (your Wi-Fi or Ethernet card).

- It receives an IP from the router, like another device on the network.
- Useful if you want other machines (or your phone) to access the VM.

```ruby
config.vm.network "public_network"
```

### 4 **Port forwarding.**

- Your PC's **8080** port redirects to the VM's **80** port.
- Ex. you launch a web server in the VM (possinginxʼ, ʼapache�...), you access it via:

```ruby
config.vm.network "forwarded_port", guest: 80, host: 8080
```

## Storagee

### Resize the Virtual Disk

```sh
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/bionic64"

  config.vm.provider "virtualbox" do |vb|
    vb.memory = "1024"
    vb.cpus = 2
    vb.customize ["modifyhd", :id, "--resize", 50000]
  end

  # Provisioning to expand the partition & filesystem automatically
  config.vm.provision "shell", inline: <<-SHELL
    sudo apt-get update -y
    sudo apt-get install -y cloud-guest-utils  # Install growpart
    sudo growpart /dev/sda 1
    sudo resize2fs /dev/sda1
  SHELL
end
```

### Resize an Existing Vagrant VM (VirtualBox)

```sh
# Step 1: Shutdown the VM
vagrant halt

# Step 2: Resize the virtual disk (e.g., 60GB)
VBoxManage modifyhd /path/to/your/disk.vdi --resize 61440  # 60GB = 61440MB

# Step 3: Start the VM again
vagrant up

# Step 4: SSH into the VM
vagrant ssh

# Step 5: Resize the partition if needed
sudo growpart /dev/sda 1

# Step 6: Resize the filesystem
sudo resize2fs /dev/sda1  # For ext4 filesystems

# Step 7: Verify the new disk size
df -h
```

## Multiple VMs

```ruby
Vagrant.configure("2") do |config|

  # VM1
  config.vm.define "web" do |web|
    web.vm.box = "ubuntu/bionic64"
    web.vm.network "private_network", ip: "192.168.33.10"
    web.vm.hostname = "web-server"
    web.vm.provider "virtualbox" do |vb|
      vb.memory = "1024"
    end
  end

  # VM2
  config.vm.define "db" do |db|
    db.vm.box = "ubuntu/bionic64"
    db.vm.network "private_network", ip: "192.168.33.11"
    db.vm.hostname = "db-server"
    db.vm.provider "virtualbox" do |vb|
      vb.memory = "512"
    end
  end
end
```

## Synced Folders

```ruby
config.vm.synced_folder "./host_folder", "/guest_folder"
config.vm.synced_folder "./data", "/vagrant_data", type: "nfs"
```

## Useful Plugins

```sh
vagrant plugin install vagrant-vbguest  # Keep Guest Additions up to date
vagrant plugin install vagrant-disksize # Resize disks
```

## Export and Share

```sh
vagrant package --output myvm.box       # Export VM
vagrant share                           # Share VM over the internet
```

## Troubleshooting

```sh
vagrant reload --provision  # Rerun provisioners
vagrant destroy -f && vagrant up  # Recreate VM
vagrant ssh -- -vvv  # Debug SSH connection
```

## Debugging

```sh
vagrant up --debug  # Enable debug mode
vagrant ssh -c "command"  # Run a command inside VM
```

## Notes

### Network Modes in a Virtual Machine

1.  **NAT (Network Address Translation)**

    - **Description**: In this mode, the VM uses the host computer's IP address to access the Internet. The VM gets an internal IP address and communicates with the outside world through the host's IP.
    - **Advantages**: Simple to set up, as no special configuration is needed on the VM to access the Internet.
    - **Usage**: This is often the default mode in many virtualization software (such as VirtualBox).

2.  **Bridged Adapter**

    - **Description**: The VM is directly connected to the physical network, as if it were another computer on the same network. This means it receives an IP address from the same DHCP server as other devices on the network.
    - **Advantages**: Allows the VM to be accessible from other machines on the local network, which is useful if you want your server to be visible on the network.
    - **Configuration**: May require additional setup if you want to assign a static IP address.

3.  **Host-only Adapter**

    - **Description**: This mode allows the VM to communicate only with the host computer and other VMs on the same network, without Internet access.
    - **Usage**: Useful for testing or development environments.
