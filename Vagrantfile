# Copyright contributors to the rhcs5-vagrant project.
# Based on vagrant-rhel8 code - Copyright (c) 2019 Andrew Garner

### Some parameters that can be adjusted

# Number of ceph-server nodes. If you change the default number of 3 you
# need to adjust the ceph-prep/config and ceph-prep/hosts files accordingly.
N = 3

# The libvirt storage pool to use
STORAGE_POOL = "default"

# Amount of Memory (RAM) per node. 8192 is recommended, but 3072 works as well.
RAM_SIZE = 8192

# IP prefix and start address for the VMs
# The ceph-admin node will get IP_PREFIX.IPSTART (default: 172.21.12.10)
# The ceph-client and ceph-server-x nodes will get subsequent addresses.
IP_PREFIX = "172.21.12."
IP_START = 10

### Do not modify the code below unless you are developing

Vagrant.require_version ">= 2.1.0" # 2.1.0 minimum required for triggers

ENV['VAGRANT_DEFAULT_PROVIDER'] = 'vmware_desktop'
user = "vagrant"
password = "vagrant"

timezone_script = %{
localectl set-keymap de
timedatectl set-timezone Europe/Berlin
}

Vagrant.configure("2") do |config|
  # config.vm.box = "roboxes/centos9s"
  config.vm.box = "edrodrig/centos10s"

  config.vm.provider :libvirt do |libvirt|
    # Do not use (user) session libvirt - VM networks will not work there on Fedora!
    libvirt.qemu_use_session = false
    libvirt.memory = RAM_SIZE
    libvirt.cpus = 2
    libvirt.disk_bus = "virtio"
    libvirt.storage_pool_name = STORAGE_POOL
    # To avoid USB device resource conflicts
    libvirt.graphics_type = "spice"
    (1..2).each do
      libvirt.redirdev :type => "spicevmc"
    end
  end
  
# === VMware Desktop provider (Fusion on macOS, Workstation/Pro on Linux/Windows) ===
  config.vm.provider "vmware_desktop" do |vmware, override|
    # Use a VMware-compatible box if the default roboxes/rhel9 does not have a vmware version
    # (Vagrant will automatically download the vmware_desktop version when needed)
    override.vm.box = "edrodrig/centos10s"   # works because roboxes publishes vmware_desktop versions too

    vmware.cpus = 2
    vmware.memory = RAM_SIZE             # respects the RAM_SIZE variable (8192 by default)
    vmware.linked_clone = true           # much faster provisioning
    vmware.gui = false                   # set to true if you want the VMware GUI
    vmware.whitelist_verified = true     # suppresses license warning on newer VMware versions
    vmware.vmx["vhv.enable"] = "true"    # enables nested virtualization (required for Ceph inside VMs)

    # Add the same number and size of extra disks that libvirt adds for OSDs
    # Only applied to ceph-server-* machines (same logic as the libvirt block)
    vmware.vmx["scsi0:1.present"] = "TRUE"
    vmware.vmx["scsi0:1.filename"] = "disk-1.vmdk"
    vmware.vmx["scsi0:1.virtualSSD"] = "1"   # treat as SSD for better performance
  end

  config.ssh.forward_agent = true
  config.ssh.insert_key = false
  config.hostmanager.enabled = true
  config.vm.synced_folder './', '/vagrant', type: 'rsync'

  # The Ceph client will be our client machine to mount volumes and interact with the cluster
  config.vm.define "ceph-client" do |client|
    client.vm.hostname = "ceph-client"
    client.vm.network "private_network", ip: IP_PREFIX+"#{IP_START + 1}"
    client.vm.provision "shell", inline: timezone_script
    client.vm.provision "ansible" do |ansible|
      ansible.playbook = "ceph-prep/client-playbook.yml"
      ansible.extra_vars = {
        node_ip: IP_PREFIX+"#{IP_START + 1}",
        user: user,
        password: password,
      }
    end
  end
 
  # We need one Ceph admin machine to manage the cluster
  config.vm.define "ceph-admin" do |admin|
    admin.vm.hostname = "ceph-admin"
    admin.vm.network "private_network", ip: IP_PREFIX+"#{IP_START}"
    admin.vm.provision "shell", inline: timezone_script
    admin.vm.provision "ansible" do |ansible|
      ansible.playbook = "ceph-prep/admin-playbook.yml"
      ansible.extra_vars = {
        node_ip: IP_PREFIX+"#{IP_START}",
        user: user,
        password: password,
      }
    end
  end
 
  # We provision three nodes to be Ceph servers
  (1..N).each do |i|
    config.vm.define "ceph-server-#{i}" do |server|
      server.vm.hostname = "ceph-server-#{i}"
      server.vm.network "private_network", ip: IP_PREFIX+"#{IP_START+i+1}"
      # Attach disks for Ceph OSDs
# ---- VMware extra disk(s) for OSDs ----
      server.vm.provider "vmware_desktop" do |vmware|
        num = 1
        (1..num).each do |d|
          disk_file = "ceph-server-#{i}-disk#{d}.vmdk"
          unless File.exist?(disk_file)
            vmware.vmx["scsi0:#{d}.present"] = "TRUE"
            vmware.vmx["scsi0:#{d}.filename"] = disk_file
            vmware.vmx["scsi0:#{d}.virtualSSD"] = "1"
          end
        end
      end  

# ---- libvirt extra disk(s) for OSDs ----    
      server.vm.provider "libvirt" do |libvirt|
        # 1 small disk
        num = 1
        (1..num).each do |disk|
          libvirt.storage :file, :size => '5G', :cache => 'none'
        end
      end
      server.vm.provision "shell", inline: timezone_script
      server.vm.provision "ansible" do |ansible|
        ansible.playbook = "ceph-prep/common-playbook.yml"
        ansible.extra_vars = {
          node_ip: IP_PREFIX+"#{IP_START+i+1}",
          user: user,
          password: password,
        }
      end
    end
  end
    
end # vagrant configure
