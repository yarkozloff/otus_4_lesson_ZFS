# Describe VMs
MACHINES = {
  # VM name "zfs"
  :"zfs" => {
              # VM box
              :box_name => "centos7",
              # VM CPU count
              :cpus => 1,
              # VM RAM size (Mb)
              :memory => 512,
              # networks
              :net => [],
              # forwarded ports
              :forwarded_port => [],
	      :provision => "installzfs.sh",
   :disks => {
	:sata1 => {
                :dfile => './sata1.vdi',
                :size => 512,
                :port => 1
	},
	:sata2 => {
                :dfile => './sata2.vdi',
                :size => 512,
                :port => 2
        },
	:sata3 => {
                :dfile => './sata3.vdi',
                :size => 512,
                :port => 3
        },
	:sata4 => {
                :dfile => './sata4.vdi',
                :size => 512,
                :port => 4	
        },
	:sata5 => {
                :dfile => './sata5.vdi',
                :size => 512,
                :port => 5
        },
	:sata6 => {
                :dfile => './sata6.vdi',
                :size => 512,
                :port => 6
        },
	:sata7 => {
                :dfile => './sata7.vdi',
                :size => 512,
                :port => 7
        },
	:sata8 => {
                :dfile => './sata8.vdi',
                :size => 512,
                :port => 8
        },	
	}
   	},
}

Vagrant.configure("2") do |config|
  config.vm.box = "centos7"
  MACHINES.each do |boxname, boxconfig|
    # Disable shared folders
    config.vm.synced_folder ".", "/vagrant", disabled: true
    # Apply VM config
    config.vm.define boxname do |box|
      # Set VM base box and hostname
      box.vm.box = boxconfig[:box_name]
      box.vm.host_name = boxname.to_s
      # Additional network config if present
      if boxconfig.key?(:net)
        boxconfig[:net].each do |ipconf|
          box.vm.network "private_network", ipconf
        end
      end
      # Port-forward config if present
      if boxconfig.key?(:forwarded_port)
        boxconfig[:forwarded_port].each do |port|
          box.vm.network "forwarded_port", port
        end
      end
      # VM resources config
      box.vm.provider "virtualbox" do |vb|
        # Set VM RAM size and CPU count
        vb.customize ["modifyvm", :id, "--memory", "512"]
	needsController = false
      boxconfig[:disks].each do |dname, dconf|
	unless File.exist?(dconf[:dfile])
	vb.customize ['createhd', '--filename', dconf[:dfile],'--variant', 'Fixed', '--size', dconf[:size]]
	needsController = true
	    end
	end
	if needsController == true
		vb.customize ["storagectl", :id, "--name", "SATA", "--add", "sata" ]
		boxconfig[:disks].each do |dname, dconf|
		vb.customize ['storageattach', :id, '--storagectl', 'SATA', '--port', dconf[:port], '--device', 0, '--type', 'hdd', '--medium', dconf[:dfile]]
    		end
	    end
	end
	 box.vm.provision "shell", path: boxconfig[:provision]
    end
  end
end

