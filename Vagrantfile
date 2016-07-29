
raise "vagrant-cumulus plugin must be installed, try $ vagrant plugin install vagrant-cumulus" unless Vagrant.has_plugin? "vagrant-cumulus"
require 'json'

$script = <<-SCRIPT
if grep -q -i 'cumulus' /etc/lsb-release &> /dev/null; then
    echo "### RUNNING CUMULUS EXTRA CONFIG ###"
    source /etc/lsb-release
    if [[ $DISTRIB_RELEASE =~ ^2.* ]]; then
        echo "  INFO: Detected a 2.5.x Based Release"
        echo "  adding fake cl-acltool..."
        echo -e "#!/bin/bash\nexit 0" > /bin/cl-acltool
        chmod 755 /bin/cl-acltool

        echo "  adding fake cl-license..."
        cat > /bin/cl-license <<'EOF'
#! /bin/bash
#-------------------------------------------------------------------------------
#
# Copyright 2013 Cumulus Networks, Inc.  All rights reserved
#

URL_RE='^http://.*/.*'

#Legacy symlink
if echo "$0" | grep -q "cl-license-install$"; then
    exec cl-license -i $*
fi

LIC_DIR="/etc/cumulus"
PERSIST_LIC_DIR="/mnt/persist/etc/cumulus"

#Print current license, if any.
if [ -z "$1" ]; then
    if [ ! -f "$LIC_DIR/.license.txt" ]; then
        echo "No license installed!" >&2
        exit 20
    fi
    cat "$LIC_DIR/.license.txt"
    exit 0
fi

#Must be root beyond this point
if (( EUID != 0 )); then
   echo "You must have root privileges to run this command." 1>&2
   exit 100
fi

#Delete license
if [ x"$1" == "x-d" ]; then
    rm -f "$LIC_DIR/.license.txt"
    rm -f "$PERSIST_LIC_DIR/.license.txt"

    echo License file uninstalled.
    exit 0
fi

function usage {
    echo "Usage: $0 (-i (license_file | URL) | -d)" >&2
    echo "    -i  Install a license, via stdin, file, or URL." >&2
    echo "    -d  Delete the current installed license." >&2
    echo >&2
    echo " cl-license prints, installs or deletes a license on this switch." >&2
}

if [ x"$1" != 'x-i' ]; then
    usage
    exit 100
fi
shift
if [ ! -f "$1" ]; then
    if [ -n "$1" ]; then
        if ! echo "$1" | grep -q "$URL_RE"; then
            usage
            echo "file $1 not found or not readable." >&2
            exit 100
        fi
    fi
fi

function clean_tmp {
    rm $1
}

if [ -z "$1" ]; then
    LIC_FILE=`mktemp lic.XXXXXX`
    trap "clean_tmp $LIC_FILE" EXIT
    echo "Paste license text here, then hit ctrl-d" >&2
    cat >$LIC_FILE
else
    if echo "$1" | grep -q "$URL_RE"; then
        LIC_FILE=`mktemp lic.XXXXXX`
        trap "clean_tmp $LIC_FILE" EXIT
        if ! wget "$1" -O $LIC_FILE; then
            echo "Couldn't download $1 via HTTP!" >&2
            exit 10
        fi
    else
        LIC_FILE="$1"
    fi
fi

/usr/sbin/switchd -lic "$LIC_FILE"
SWITCHD_RETCODE=$?
if [ $SWITCHD_RETCODE -eq 99 ]; then
    more /usr/share/cumulus/EULA.txt
    echo "I (on behalf of the entity who will be using the software) accept"
    read -p "and agree to the EULA  (yes/no): "
    if [ "$REPLY" != "yes" -a "$REPLY" != "y" ]; then
        echo EULA not agreed to, aborting. >&2
        exit 2
    fi
elif [ $SWITCHD_RETCODE -ne 0 ]; then
    echo '******************************' >&2
    echo ERROR: License file not valid. >&2
    echo ERROR: No license installed. >&2
    echo '******************************' >&2
    exit 1
fi

mkdir -p "$LIC_DIR"
cp "$LIC_FILE" "$LIC_DIR/.license.txt"
chmod 644 "$LIC_DIR/.license.txt"

mkdir -p "$PERSIST_LIC_DIR"
cp "$LIC_FILE" "$PERSIST_LIC_DIR/.license.txt"
chmod 644 "$PERSIST_LIC_DIR/.license.txt"

echo License file installed.
echo Reboot to enable functionality.
EOF
        chmod 755 /bin/cl-license

        echo "  Disabling default remap on Cumulus VX..."
        mv -v /etc/init.d/rename_eth_swp /etc/init.d/rename_eth_swp.backup

        echo "  Replacing fake switchd"
        rm -rf /usr/bin/switchd
        cat > /usr/sbin/switchd <<'EOF'
#!/bin/bash
PIDFILE=$1
LICENSE_FILE=/etc/cumulus/.license
RC=0

# Make sure we weren't invoked with "-lic"
if [ "$PIDFILE" == "-lic" ]; then
  if [ "$2" != "" ]; then
        LICENSE_FILE=$2
  fi
  if [ ! -e $LICENSE_FILE ]; then
    echo "No license file." >&2
    RC=1
  fi
  RC=0
else
  tail -f /dev/null & CPID=$!
  echo -n $CPID > $PIDFILE
  wait $CPID
fi

exit $RC
EOF
        chmod 755 /usr/sbin/switchd

        cat > /etc/init.d/switchd <<'EOF'
#! /bin/bash
### BEGIN INIT INFO
# Provides:          switchd
# Required-Start:
# Required-Stop:
# Should-Start:
# Should-Stop:
# X-Start-Before
# Default-Start:     S
# Default-Stop:
# Short-Description: Controls fake switchd process
### END INIT INFO

# Author: Kristian Van Der Vliet <kristian@cumulusnetworks.com>
#
# Please remove the "Author" lines above and replace them
# with your own name if you copy and modify this script.

PATH=/sbin:/bin:/usr/bin

NAME=switchd
SCRIPTNAME=/etc/init.d/$NAME
PIDFILE=/var/run/switchd.pid

. /lib/init/vars.sh
. /lib/lsb/init-functions

do_start() {
  echo "[ ok ] Starting Cumulus Networks switch chip daemon: switchd"
  /usr/sbin/switchd $PIDFILE 2>/dev/null &
}

do_stop() {
  if [ -e $PIDFILE ];then
    kill -TERM $(cat $PIDFILE)
    rm $PIDFILE
  fi
}

do_status() {
  if [ -e $PIDFILE ];then
    echo "[ ok ] switchd is running."
  else
    echo "[FAIL] switchd is not running ... failed!" >&2
    exit 3
  fi
}

case "$1" in
  start|"")
	log_action_begin_msg "Starting switchd"
	do_start
	log_action_end_msg 0
	;;
  stop)
  log_action_begin_msg "Stopping switchd"
  do_stop
  log_action_end_msg 0
  ;;
  status)
  do_status
  ;;
  restart)
	log_action_begin_msg "Re-starting switchd"
  do_stop
	do_start
	log_action_end_msg 0
	;;
  reload|force-reload)
	echo "Error: argument '$1' not supported" >&2
	exit 3
	;;
  *)
	echo "Usage: $SCRIPTNAME [start|stop|restart|status]" >&2
	exit 3
	;;
esac

:
EOF
        chmod 755 /etc/init.d/switchd
        reboot

    elif [[ $DISTRIB_RELEASE =~ ^3.* ]]; then
        echo "  INFO: Detected a 3.x Based Release"

        echo "  Disabling default remap on Cumulus VX..."
        mv -v /etc/hw_init.d/S10rename_eth_swp.sh /etc/S10rename_eth_swp.sh.backup
        reboot
    fi
    echo "### DONE ###"
else
    reboot
fi
SCRIPT



Vagrant.configure("2") do |config|
  wbid = 1
  config.vm.provider "virtualbox" do |v|
    v.gui=false
  end




  config.vm.define "workbench" do |device|
      device.vm.provider "virtualbox" do |v|
        v.name = "workbench"
        v.memory = 1024
      end
      device.vm.hostname = "workbench"
      device.vm.box = "boxcutter/ubuntu1404"


    device.vm.network "private_network", virtualbox__intnet: "net_workbench", auto_config: false


      device.vm.synced_folder ".", "/vagrant", disabled: true
      device.vm.provision :shell , inline: "sudo sed -i 's/sleep [0-9]*/sleep 1/' /etc/init/failsafe.conf"
      device.vm.provision :shell , path: "./helper_scripts/oob_server_config.sh"
      device.vm.provision "file", source: "./helper_scripts/rename_eth_swp", destination: "/home/vagrant/rename_eth_swp"
      device.vm.provision "file", source: "./helper_scripts/autogenerated/oob-mgmt-server_remap_eth", destination: "/home/vagrant/remap_eth"
      device.vm.provision :shell , inline: "mv /home/vagrant/rename_eth_swp /etc/init.d/rename_eth_swp"
      device.vm.provision :shell , inline: "mv /home/vagrant/remap_eth /etc/default/remap_eth"
      device.vm.provision :shell , inline: "chmod 755 /etc/init.d/rename_eth_swp"
      device.vm.provision :shell , inline: "sudo sed -i '/exit 0/d' /etc/rc.local "
      device.vm.provision :shell , inline: "sudo echo '/etc/init.d/rename_eth_swp start' >> /etc/rc.local "
      device.vm.provision :shell , inline: "sudo echo 'exit 0' >> /etc/rc.local "
      device.vm.provision :shell , inline: "/etc/init.d/rename_eth_swp verbose"

      device.vm.provider "virtualbox" do |vbox|

        vbox.customize ['modifyvm', :id, '--nicpromisc2', 'allow-vms']


        vbox.customize ["modifyvm", :id, "--nictype1", "virtio"]

        vbox.customize ["modifyvm", :id, "--nictype2", "virtio"]

      end

      device.vm.provision "ansible" do |ansible|
        ansible.playbook = "./wbench/wbench.yml"
        ansible.extra_vars = {wbench_hosts: {



                spine01: {ip: "192.168.0.21", mac: "A0:00:00:00:00:21"},



                spine02: {ip: "192.168.0.22", mac: "A0:00:00:00:00:22"},



                spine03: {ip: "192.168.0.23", mac: "A0:00:00:00:00:23"},



                spine04: {ip: "192.168.0.24", mac: "A0:00:00:00:00:24"},



                leaf01: {ip: "192.168.0.11", mac: "A0:00:00:00:00:11"},



                leaf02: {ip: "192.168.0.12", mac: "A0:00:00:00:00:12"},



                leaf03: {ip: "192.168.0.13", mac: "A0:00:00:00:00:13"},



                leaf04: {ip: "192.168.0.14", mac: "A0:00:00:00:00:14"},



                leaf05: {ip: "192.168.0.15", mac: "A0:00:00:00:00:15"},



                leaf06: {ip: "192.168.0.16", mac: "A0:00:00:00:00:16"},



                leaf07: {ip: "192.168.0.17", mac: "A0:00:00:00:00:17"},



                leaf08: {ip: "192.168.0.18", mac: "A0:00:00:00:00:18"},

                            }}
      end
  end



config.vm.define "oob-mgmt-switch" do |device|
    device.vm.provider "virtualbox" do |v|
      v.name = "oob-mgmt-switch"
      v.memory = 256
    end
    device.vm.hostname = "oob-mgmt-switch"
    device.vm.box = "cumulus-vx-3.0.0"

        device.vm.network "private_network", virtualbox__intnet: "net_workbench", auto_config: false , :mac => "A00000000000"

        device.vm.network "private_network", virtualbox__intnet: "net_l1_eth0", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l2_eth0", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l3_eth0", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l4_eth0", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l5_eth0", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l6_eth0", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l7_eth0", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l8_eth0", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_s1_eth0", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_s2_eth0", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_s3_eth0", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_s4_eth0", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_#{wbid}_57", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_#{wbid}_58", auto_config: false


    # Disabling the default synced folder
    device.vm.synced_folder ".", "/vagrant", disabled: true
#    device.vm.provision :shell , path: "./helper_scripts/oob_switch_config.sh"
#    device.vm.provision "file", source: "./helper_scripts/rename_eth_swp", destination: "/home/vagrant/rename_eth_swp"
#    device.vm.provision "file", source: "./helper_scripts/autogenerated/oob-mgmt-switch_remap_eth", destination: "/home/vagrant/remap_eth"
#    device.vm.provision :shell , inline: "mv /home/vagrant/rename_eth_swp /etc/init.d/rename_eth_swp"
#    device.vm.provision :shell , inline: "mv /home/vagrant/remap_eth /etc/default/remap_eth"
#    device.vm.provision :shell , inline: "chmod 755 /etc/init.d/rename_eth_swp"
#    device.vm.provision :shell , inline: "/etc/init.d/rename_eth_swp verbose"

    device.vm.provider "virtualbox" do |vbox|
        vbox.customize ['modifyvm', :id, '--nicpromisc2', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc3', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc4', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc5', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc6', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc7', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc8', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc9', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc10', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc11', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc12', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc13', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc14', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc15', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc16', 'allow-vms']

        vbox.customize ["modifyvm", :id, "--nictype1", "virtio"]

    end

        # Fixes "stdin: is not a tty" message --> https://github.com/mitchellh/vagrant/issues/1673
        device.vm.provision :shell , inline: "(grep -q -E '^mesg n$' /root/.profile && sed -i 's/^mesg n$/tty -s \\&\\& mesg n/g' /root/.profile && echo 'Ignore the previous error \"stdin: is not a tty\" -- fixing this now...') || exit 0;"

        # Run Any Extra Config
        device.vm.provision :shell , path: "./helper_scripts/config_switch.sh"

        # Apply the interface re-map
        device.vm.provision "file", source: "./helper_scripts/apply_udev.py", destination: "/home/vagrant/apply_udev.py"
        device.vm.provision :shell , inline: "chmod 755 /home/vagrant/apply_udev.py"
        device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -a A0:00:00:00:00:21 eth0"
        device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -vm "
        device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -s"
        device.vm.provision :shell , :inline => $script

end






config.vm.define "spine01" do |device|
    device.vm.provider "virtualbox" do |v|
      v.name = "spine01"
      v.memory = 512
    end
    device.vm.hostname = "spine01"
    device.vm.box = "cumulus-vx-3.0.0"


        device.vm.network "private_network", virtualbox__intnet: "net_s1_eth0", auto_config: false , :mac => "A00000000021"

        device.vm.network "private_network", virtualbox__intnet: "net_l1s1", auto_config: false , :mac => "44383900005C"

        device.vm.network "private_network", virtualbox__intnet: "net_l2s1", auto_config: false , :mac => "44383900002F"

        device.vm.network "private_network", virtualbox__intnet: "net_l3s1", auto_config: false , :mac => "443839000058"

        device.vm.network "private_network", virtualbox__intnet: "net_l4s1", auto_config: false , :mac => "443839000044"

        device.vm.network "private_network", virtualbox__intnet: "net_l5s1", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l6s1", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l7s1", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l8s1", auto_config: false


    # Disabling the default synced folder
    device.vm.synced_folder ".", "/vagrant", disabled: true
#    device.vm.provision :shell , path: "./helper_scripts/pre_config.sh"
#    device.vm.provision "file", source: "./helper_scripts/rename_eth_swp", destination: "/home/vagrant/rename_eth_swp"
#    device.vm.provision "file", source: "./helper_scripts/autogenerated/spine01_remap_eth", destination: "/home/vagrant/remap_eth"
#    device.vm.provision :shell , inline: "mv /home/vagrant/rename_eth_swp /etc/init.d/rename_eth_swp"
#    device.vm.provision :shell , inline: "mv /home/vagrant/remap_eth /etc/default/remap_eth"
#    device.vm.provision :shell , inline: "chmod 755 /etc/init.d/rename_eth_swp"
#    device.vm.provision :shell , inline: "/etc/init.d/rename_eth_swp verbose"

    device.vm.provider "virtualbox" do |vbox|
        vbox.customize ['modifyvm', :id, '--nicpromisc2', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc3', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc4', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc5', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc6', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc7', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc8', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc9', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc10', 'allow-vms']

        vbox.customize ["modifyvm", :id, "--nictype1", "virtio"]

    end

        # Fixes "stdin: is not a tty" message --> https://github.com/mitchellh/vagrant/issues/1673
        device.vm.provision :shell , inline: "(grep -q -E '^mesg n$' /root/.profile && sed -i 's/^mesg n$/tty -s \\&\\& mesg n/g' /root/.profile && echo 'Ignore the previous error \"stdin: is not a tty\" -- fixing this now...') || exit 0;"

        # Run Any Extra Config
        device.vm.provision :shell , path: "./helper_scripts/config_switch.sh"

        # Apply the interface re-map
        device.vm.provision "file", source: "./helper_scripts/apply_udev.py", destination: "/home/vagrant/apply_udev.py"
        device.vm.provision :shell , inline: "chmod 755 /home/vagrant/apply_udev.py"
        device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -a A0:00:00:00:00:21 eth0"
        device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -a 44383900005C swp1"
        device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -a 44383900002F swp2"
        device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -a 443839000058 swp3"
        device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -a 443839000044 swp4"
        device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -vm "
        device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -s"
        device.vm.provision :shell , :inline => $script

end




config.vm.define "spine02" do |device|
    device.vm.provider "virtualbox" do |v|
      v.name = "spine02"
      v.memory = 512
    end
    device.vm.hostname = "spine02"
    device.vm.box = "cumulus-vx-3.0.0"


        device.vm.network "private_network", virtualbox__intnet: "net_s2_eth0", auto_config: false , :mac => "A00000000022"

        device.vm.network "private_network", virtualbox__intnet: "net_l1s2", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l2s2", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l3s2", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l4s2", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l5s2", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l6s2", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l7s2", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l8s2", auto_config: false


    # Disabling the default synced folder
    device.vm.synced_folder ".", "/vagrant", disabled: true
#    device.vm.provision :shell , path: "./helper_scripts/pre_config.sh"
#    device.vm.provision "file", source: "./helper_scripts/rename_eth_swp", destination: "/home/vagrant/rename_eth_swp"
#    device.vm.provision "file", source: "./helper_scripts/autogenerated/spine02_remap_eth", destination: "/home/vagrant/remap_eth"
#    device.vm.provision :shell , inline: "mv /home/vagrant/rename_eth_swp /etc/init.d/rename_eth_swp"
#    device.vm.provision :shell , inline: "mv /home/vagrant/remap_eth /etc/default/remap_eth"
#    device.vm.provision :shell , inline: "chmod 755 /etc/init.d/rename_eth_swp"
#    device.vm.provision :shell , inline: "/etc/init.d/rename_eth_swp verbose"

    device.vm.provider "virtualbox" do |vbox|
        vbox.customize ['modifyvm', :id, '--nicpromisc2', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc3', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc4', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc5', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc6', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc7', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc8', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc9', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc10', 'allow-vms']

        vbox.customize ["modifyvm", :id, "--nictype1", "virtio"]

    end

        # Fixes "stdin: is not a tty" message --> https://github.com/mitchellh/vagrant/issues/1673
        device.vm.provision :shell , inline: "(grep -q -E '^mesg n$' /root/.profile && sed -i 's/^mesg n$/tty -s \\&\\& mesg n/g' /root/.profile && echo 'Ignore the previous error \"stdin: is not a tty\" -- fixing this now...') || exit 0;"

        # Run Any Extra Config
        device.vm.provision :shell , path: "./helper_scripts/config_switch.sh"

        # Apply the interface re-map
        device.vm.provision "file", source: "./helper_scripts/apply_udev.py", destination: "/home/vagrant/apply_udev.py"
        device.vm.provision :shell , inline: "chmod 755 /home/vagrant/apply_udev.py"
        device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -a A0:00:00:00:00:22 eth0"
        device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -vm "
        device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -s"
        device.vm.provision :shell , :inline => $script

end



config.vm.define "spine03" do |device|
    device.vm.provider "virtualbox" do |v|
      v.name = "spine03"
      v.memory = 512
    end
    device.vm.hostname = "spine03"
    device.vm.box = "cumulus-vx-3.0.0"


        device.vm.network "private_network", virtualbox__intnet: "net_s3_eth0", auto_config: false , :mac => "A00000000023"

        device.vm.network "private_network", virtualbox__intnet: "net_l1s3", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l2s3", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l3s3", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l4s3", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l5s3", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l6s3", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l7s3", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l8s3", auto_config: false


    # Disabling the default synced folder
    device.vm.synced_folder ".", "/vagrant", disabled: true
#    device.vm.provision :shell , path: "./helper_scripts/pre_config.sh"
#    device.vm.provision "file", source: "./helper_scripts/rename_eth_swp", destination: "/home/vagrant/rename_eth_swp"
#    device.vm.provision "file", source: "./helper_scripts/autogenerated/spine03_remap_eth", destination: "/home/vagrant/remap_eth"
#    device.vm.provision :shell , inline: "mv /home/vagrant/rename_eth_swp /etc/init.d/rename_eth_swp"
#    device.vm.provision :shell , inline: "mv /home/vagrant/remap_eth /etc/default/remap_eth"
#    device.vm.provision :shell , inline: "chmod 755 /etc/init.d/rename_eth_swp"
#    device.vm.provision :shell , inline: "/etc/init.d/rename_eth_swp verbose"

    device.vm.provider "virtualbox" do |vbox|
        vbox.customize ['modifyvm', :id, '--nicpromisc2', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc3', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc4', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc5', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc6', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc7', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc8', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc9', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc10', 'allow-vms']

        vbox.customize ["modifyvm", :id, "--nictype1", "virtio"]

    end

        # Fixes "stdin: is not a tty" message --> https://github.com/mitchellh/vagrant/issues/1673
        device.vm.provision :shell , inline: "(grep -q -E '^mesg n$' /root/.profile && sed -i 's/^mesg n$/tty -s \\&\\& mesg n/g' /root/.profile && echo 'Ignore the previous error \"stdin: is not a tty\" -- fixing this now...') || exit 0;"

        # Run Any Extra Config
        device.vm.provision :shell , path: "./helper_scripts/config_switch.sh"

        # Apply the interface re-map
        device.vm.provision "file", source: "./helper_scripts/apply_udev.py", destination: "/home/vagrant/apply_udev.py"
        device.vm.provision :shell , inline: "chmod 755 /home/vagrant/apply_udev.py"
        device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -a A0:00:00:00:00:23 eth0"
        device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -vm "
        device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -s"
        device.vm.provision :shell , :inline => $script

end


config.vm.define "spine04" do |device|
    device.vm.provider "virtualbox" do |v|
      v.name = "spine04"
      v.memory = 512
    end
    device.vm.hostname = "spine04"
    device.vm.box = "cumulus-vx-3.0.0"


        device.vm.network "private_network", virtualbox__intnet: "net_s4_eth0", auto_config: false , :mac => "A00000000024"

        device.vm.network "private_network", virtualbox__intnet: "net_l1s4", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l2s4", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l3s4", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l4s4", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l5s4", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l6s4", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l7s4", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l8s4", auto_config: false


    # Disabling the default synced folder
    device.vm.synced_folder ".", "/vagrant", disabled: true
#    device.vm.provision :shell , path: "./helper_scripts/pre_config.sh"
#    device.vm.provision "file", source: "./helper_scripts/rename_eth_swp", destination: "/home/vagrant/rename_eth_swp"
#    device.vm.provision "file", source: "./helper_scripts/autogenerated/spine04_remap_eth", destination: "/home/vagrant/remap_eth"
#    device.vm.provision :shell , inline: "mv /home/vagrant/rename_eth_swp /etc/init.d/rename_eth_swp"
#    device.vm.provision :shell , inline: "mv /home/vagrant/remap_eth /etc/default/remap_eth"
#    device.vm.provision :shell , inline: "chmod 755 /etc/init.d/rename_eth_swp"
#    device.vm.provision :shell , inline: "/etc/init.d/rename_eth_swp verbose"

    device.vm.provider "virtualbox" do |vbox|
        vbox.customize ['modifyvm', :id, '--nicpromisc2', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc3', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc4', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc5', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc6', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc7', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc8', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc9', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc10', 'allow-vms']

        vbox.customize ["modifyvm", :id, "--nictype1", "virtio"]

    end

        # Fixes "stdin: is not a tty" message --> https://github.com/mitchellh/vagrant/issues/1673
        device.vm.provision :shell , inline: "(grep -q -E '^mesg n$' /root/.profile && sed -i 's/^mesg n$/tty -s \\&\\& mesg n/g' /root/.profile && echo 'Ignore the previous error \"stdin: is not a tty\" -- fixing this now...') || exit 0;"

        # Run Any Extra Config
        device.vm.provision :shell , path: "./helper_scripts/config_switch.sh"

        # Apply the interface re-map
        device.vm.provision "file", source: "./helper_scripts/apply_udev.py", destination: "/home/vagrant/apply_udev.py"
        device.vm.provision :shell , inline: "chmod 755 /home/vagrant/apply_udev.py"
        device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -a A0:00:00:00:00:24 eth0"
        device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -vm "
        device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -s"
        device.vm.provision :shell , :inline => $script

end




config.vm.define "leaf01" do |device|
    device.vm.provider "virtualbox" do |v|
      v.name = "leaf01"
      v.memory = 512
    end
    device.vm.hostname = "leaf01"
    device.vm.box = "cumulus-vx-3.0.0"


        device.vm.network "private_network", virtualbox__intnet: "net_l1_eth0", auto_config: false , :mac => "A00000000011"

        device.vm.network "private_network", virtualbox__intnet: "net_#{wbid}_16", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_#{wbid}_18", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_#{wbid}_28", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_#{wbid}_28", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_#{wbid}_34", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_#{wbid}_34", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l1s1", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l1s2", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l1s3", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l1s4", auto_config: false


    # Disabling the default synced folder
    device.vm.synced_folder ".", "/vagrant", disabled: true
#    device.vm.provision :shell , path: "./helper_scripts/pre_config.sh"
#    device.vm.provision "file", source: "./helper_scripts/rename_eth_swp", destination: "/home/vagrant/rename_eth_swp"
#    device.vm.provision "file", source: "./helper_scripts/autogenerated/leaf01_remap_eth", destination: "/home/vagrant/remap_eth"
#    device.vm.provision :shell , inline: "mv /home/vagrant/rename_eth_swp /etc/init.d/rename_eth_swp"
#    device.vm.provision :shell , inline: "mv /home/vagrant/remap_eth /etc/default/remap_eth"
#    device.vm.provision :shell , inline: "chmod 755 /etc/init.d/rename_eth_swp"
#    device.vm.provision :shell , inline: "/etc/init.d/rename_eth_swp verbose"

    device.vm.provider "virtualbox" do |vbox|
        vbox.customize ['modifyvm', :id, '--nicpromisc2', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc3', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc4', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc5', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc6', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc7', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc8', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc9', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc10', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc11', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc12', 'allow-vms']

        vbox.customize ["modifyvm", :id, "--nictype1", "virtio"]

    end

        # Fixes "stdin: is not a tty" message --> https://github.com/mitchellh/vagrant/issues/1673
        device.vm.provision :shell , inline: "(grep -q -E '^mesg n$' /root/.profile && sed -i 's/^mesg n$/tty -s \\&\\& mesg n/g' /root/.profile && echo 'Ignore the previous error \"stdin: is not a tty\" -- fixing this now...') || exit 0;"

        # Run Any Extra Config
        device.vm.provision :shell , path: "./helper_scripts/config_switch.sh"

        # Apply the interface re-map
        device.vm.provision "file", source: "./helper_scripts/apply_udev.py", destination: "/home/vagrant/apply_udev.py"
        device.vm.provision :shell , inline: "chmod 755 /home/vagrant/apply_udev.py"
        device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -a A0:00:00:00:00:11 eth0"
        device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -vm "
        device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -s"
        device.vm.provision :shell , :inline => $script

end




config.vm.define "leaf02" do |device|
    device.vm.provider "virtualbox" do |v|
      v.name = "leaf02"
      v.memory = 512
    end
    device.vm.hostname = "leaf02"
    device.vm.box = "cumulus-vx-3.0.0"


        device.vm.network "private_network", virtualbox__intnet: "net_l2_eth0", auto_config: false , :mac => "A00000000012"

        device.vm.network "private_network", virtualbox__intnet: "net_#{wbid}_17", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_#{wbid}_19", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_#{wbid}_29", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_#{wbid}_29", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_#{wbid}_35", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_#{wbid}_35", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l2s1", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l2s2", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l2s3", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l2s4", auto_config: false


    # Disabling the default synced folder
    device.vm.synced_folder ".", "/vagrant", disabled: true
#    device.vm.provision :shell , path: "./helper_scripts/pre_config.sh"
#    device.vm.provision "file", source: "./helper_scripts/rename_eth_swp", destination: "/home/vagrant/rename_eth_swp"
#    device.vm.provision "file", source: "./helper_scripts/autogenerated/leaf02_remap_eth", destination: "/home/vagrant/remap_eth"
#    device.vm.provision :shell , inline: "mv /home/vagrant/rename_eth_swp /etc/init.d/rename_eth_swp"
#    device.vm.provision :shell , inline: "mv /home/vagrant/remap_eth /etc/default/remap_eth"
#    device.vm.provision :shell , inline: "chmod 755 /etc/init.d/rename_eth_swp"
#    device.vm.provision :shell , inline: "/etc/init.d/rename_eth_swp verbose"

    device.vm.provider "virtualbox" do |vbox|
        vbox.customize ['modifyvm', :id, '--nicpromisc2', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc3', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc4', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc5', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc6', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc7', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc8', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc9', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc10', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc11', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc12', 'allow-vms']

        vbox.customize ["modifyvm", :id, "--nictype1", "virtio"]

    end

        # Fixes "stdin: is not a tty" message --> https://github.com/mitchellh/vagrant/issues/1673
        device.vm.provision :shell , inline: "(grep -q -E '^mesg n$' /root/.profile && sed -i 's/^mesg n$/tty -s \\&\\& mesg n/g' /root/.profile && echo 'Ignore the previous error \"stdin: is not a tty\" -- fixing this now...') || exit 0;"

        # Run Any Extra Config
        device.vm.provision :shell , path: "./helper_scripts/config_switch.sh"

        # Apply the interface re-map
        device.vm.provision "file", source: "./helper_scripts/apply_udev.py", destination: "/home/vagrant/apply_udev.py"
        device.vm.provision :shell , inline: "chmod 755 /home/vagrant/apply_udev.py"
        device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -a A0:00:00:00:00:12 eth0"
        device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -vm "
        device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -s"
        device.vm.provision :shell , :inline => $script

end




config.vm.define "leaf03" do |device|
    device.vm.provider "virtualbox" do |v|
      v.name = "leaf03"
      v.memory = 512
    end
    device.vm.hostname = "leaf03"
    device.vm.box = "cumulus-vx-3.0.0"


        device.vm.network "private_network", virtualbox__intnet: "net_l3_eth0", auto_config: false , :mac => "A00000000013"

        device.vm.network "private_network", virtualbox__intnet: "net_#{wbid}_20", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_#{wbid}_22", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_#{wbid}_30", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_#{wbid}_30", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_#{wbid}_36", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_#{wbid}_36", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l3s1", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l3s2", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l3s3", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l3s4", auto_config: false


    # Disabling the default synced folder
    device.vm.synced_folder ".", "/vagrant", disabled: true
#    device.vm.provision :shell , path: "./helper_scripts/pre_config.sh"
#    device.vm.provision "file", source: "./helper_scripts/rename_eth_swp", destination: "/home/vagrant/rename_eth_swp"
#    device.vm.provision "file", source: "./helper_scripts/autogenerated/leaf03_remap_eth", destination: "/home/vagrant/remap_eth"
#    device.vm.provision :shell , inline: "mv /home/vagrant/rename_eth_swp /etc/init.d/rename_eth_swp"
#    device.vm.provision :shell , inline: "mv /home/vagrant/remap_eth /etc/default/remap_eth"
#    device.vm.provision :shell , inline: "chmod 755 /etc/init.d/rename_eth_swp"
#    device.vm.provision :shell , inline: "/etc/init.d/rename_eth_swp verbose"

    device.vm.provider "virtualbox" do |vbox|
        vbox.customize ['modifyvm', :id, '--nicpromisc2', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc3', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc4', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc5', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc6', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc7', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc8', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc9', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc10', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc11', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc12', 'allow-vms']

        vbox.customize ["modifyvm", :id, "--nictype1", "virtio"]

    end

        # Fixes "stdin: is not a tty" message --> https://github.com/mitchellh/vagrant/issues/1673
        device.vm.provision :shell , inline: "(grep -q -E '^mesg n$' /root/.profile && sed -i 's/^mesg n$/tty -s \\&\\& mesg n/g' /root/.profile && echo 'Ignore the previous error \"stdin: is not a tty\" -- fixing this now...') || exit 0;"

        # Run Any Extra Config
        device.vm.provision :shell , path: "./helper_scripts/config_switch.sh"

        # Apply the interface re-map
        device.vm.provision "file", source: "./helper_scripts/apply_udev.py", destination: "/home/vagrant/apply_udev.py"
        device.vm.provision :shell , inline: "chmod 755 /home/vagrant/apply_udev.py"
        device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -a A0:00:00:00:00:13 eth0"
        device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -vm "
        device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -s"
        device.vm.provision :shell , :inline => $script

end




config.vm.define "leaf04" do |device|
    device.vm.provider "virtualbox" do |v|
      v.name = "leaf04"
      v.memory = 512
    end
    device.vm.hostname = "leaf04"
    device.vm.box = "cumulus-vx-3.0.0"


        device.vm.network "private_network", virtualbox__intnet: "net_l4_eth0", auto_config: false , :mac => "A00000000014"

        device.vm.network "private_network", virtualbox__intnet: "net_#{wbid}_21", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_#{wbid}_23", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_#{wbid}_31", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_#{wbid}_31", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_#{wbid}_37", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_#{wbid}_37", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l4s1", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l4s2", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l4s3", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l4s4", auto_config: false


    # Disabling the default synced folder
    device.vm.synced_folder ".", "/vagrant", disabled: true
#    device.vm.provision :shell , path: "./helper_scripts/pre_config.sh"
#    device.vm.provision "file", source: "./helper_scripts/rename_eth_swp", destination: "/home/vagrant/rename_eth_swp"
#    device.vm.provision "file", source: "./helper_scripts/autogenerated/leaf04_remap_eth", destination: "/home/vagrant/remap_eth"
#    device.vm.provision :shell , inline: "mv /home/vagrant/rename_eth_swp /etc/init.d/rename_eth_swp"
#    device.vm.provision :shell , inline: "mv /home/vagrant/remap_eth /etc/default/remap_eth"
#    device.vm.provision :shell , inline: "chmod 755 /etc/init.d/rename_eth_swp"
#    device.vm.provision :shell , inline: "/etc/init.d/rename_eth_swp verbose"

    device.vm.provider "virtualbox" do |vbox|
        vbox.customize ['modifyvm', :id, '--nicpromisc2', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc3', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc4', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc5', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc6', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc7', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc8', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc9', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc10', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc11', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc12', 'allow-vms']

        vbox.customize ["modifyvm", :id, "--nictype1", "virtio"]

    end

        # Fixes "stdin: is not a tty" message --> https://github.com/mitchellh/vagrant/issues/1673
        device.vm.provision :shell , inline: "(grep -q -E '^mesg n$' /root/.profile && sed -i 's/^mesg n$/tty -s \\&\\& mesg n/g' /root/.profile && echo 'Ignore the previous error \"stdin: is not a tty\" -- fixing this now...') || exit 0;"

        # Run Any Extra Config
        device.vm.provision :shell , path: "./helper_scripts/config_switch.sh"

        # Apply the interface re-map
        device.vm.provision "file", source: "./helper_scripts/apply_udev.py", destination: "/home/vagrant/apply_udev.py"
        device.vm.provision :shell , inline: "chmod 755 /home/vagrant/apply_udev.py"
        device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -a A0:00:00:00:00:14 eth0"
        device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -vm "
        device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -s"
        device.vm.provision :shell , :inline => $script

  end





config.vm.define "leaf05" do |device|
    device.vm.provider "virtualbox" do |v|
      v.name = "leaf05"
      v.memory = 512
    end
    device.vm.hostname = "leaf05"
    device.vm.box = "cumulus-vx-3.0.0"


        device.vm.network "private_network", virtualbox__intnet: "net_l5_eth0", auto_config: false , :mac => "A00000000015"

        device.vm.network "private_network", virtualbox__intnet: "net_#{wbid}_21", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_#{wbid}_23", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_#{wbid}_31", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_#{wbid}_31", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_#{wbid}_37", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_#{wbid}_37", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l5s1", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l5s2", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l5s3", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l5s4", auto_config: false


    # Disabling the default synced folder
    device.vm.synced_folder ".", "/vagrant", disabled: true
#    device.vm.provision :shell , path: "./helper_scripts/pre_config.sh"
#    device.vm.provision "file", source: "./helper_scripts/rename_eth_swp", destination: "/home/vagrant/rename_eth_swp"
#    device.vm.provision "file", source: "./helper_scripts/autogenerated/leaf05_remap_eth", destination: "/home/vagrant/remap_eth"
#    device.vm.provision :shell , inline: "mv /home/vagrant/rename_eth_swp /etc/init.d/rename_eth_swp"
#    device.vm.provision :shell , inline: "mv /home/vagrant/remap_eth /etc/default/remap_eth"
#    device.vm.provision :shell , inline: "chmod 755 /etc/init.d/rename_eth_swp"
#    device.vm.provision :shell , inline: "/etc/init.d/rename_eth_swp verbose"

    device.vm.provider "virtualbox" do |vbox|
        vbox.customize ['modifyvm', :id, '--nicpromisc2', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc3', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc4', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc5', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc6', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc7', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc8', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc9', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc10', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc11', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc12', 'allow-vms']

        vbox.customize ["modifyvm", :id, "--nictype1", "virtio"]

    end

        # Fixes "stdin: is not a tty" message --> https://github.com/mitchellh/vagrant/issues/1673
        device.vm.provision :shell , inline: "(grep -q -E '^mesg n$' /root/.profile && sed -i 's/^mesg n$/tty -s \\&\\& mesg n/g' /root/.profile && echo 'Ignore the previous error \"stdin: is not a tty\" -- fixing this now...') || exit 0;"

        # Run Any Extra Config
        device.vm.provision :shell , path: "./helper_scripts/config_switch.sh"

        # Apply the interface re-map
        device.vm.provision "file", source: "./helper_scripts/apply_udev.py", destination: "/home/vagrant/apply_udev.py"
        device.vm.provision :shell , inline: "chmod 755 /home/vagrant/apply_udev.py"
        device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -a A0:00:00:00:00:15 eth0"
        device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -vm "
        device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -s"
        device.vm.provision :shell , :inline => $script

  end





  config.vm.define "leaf06" do |device|
    device.vm.provider "virtualbox" do |v|
      v.name = "leaf06"
      v.memory = 512
    end
    device.vm.hostname = "leaf06"
    device.vm.box = "cumulus-vx-3.0.0"


        device.vm.network "private_network", virtualbox__intnet: "net_l6_eth0", auto_config: false , :mac => "A00000000016"

        device.vm.network "private_network", virtualbox__intnet: "net_#{wbid}_21", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_#{wbid}_23", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_#{wbid}_31", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_#{wbid}_31", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_#{wbid}_37", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_#{wbid}_37", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l6s1", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l6s2", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l6s3", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l6s4", auto_config: false


    # Disabling the default synced folder
    device.vm.synced_folder ".", "/vagrant", disabled: true
#    device.vm.provision :shell , path: "./helper_scripts/pre_config.sh"
#    device.vm.provision "file", source: "./helper_scripts/rename_eth_swp", destination: "/home/vagrant/rename_eth_swp"
#    device.vm.provision "file", source: "./helper_scripts/autogenerated/leaf06_remap_eth", destination: "/home/vagrant/remap_eth"
#    device.vm.provision :shell , inline: "mv /home/vagrant/rename_eth_swp /etc/init.d/rename_eth_swp"
#    device.vm.provision :shell , inline: "mv /home/vagrant/remap_eth /etc/default/remap_eth"
#    device.vm.provision :shell , inline: "chmod 755 /etc/init.d/rename_eth_swp"
#    device.vm.provision :shell , inline: "/etc/init.d/rename_eth_swp verbose"

    device.vm.provider "virtualbox" do |vbox|
        vbox.customize ['modifyvm', :id, '--nicpromisc2', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc3', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc4', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc5', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc6', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc7', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc8', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc9', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc10', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc11', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc12', 'allow-vms']

        vbox.customize ["modifyvm", :id, "--nictype1", "virtio"]

    end

        # Fixes "stdin: is not a tty" message --> https://github.com/mitchellh/vagrant/issues/1673
        device.vm.provision :shell , inline: "(grep -q -E '^mesg n$' /root/.profile && sed -i 's/^mesg n$/tty -s \\&\\& mesg n/g' /root/.profile && echo 'Ignore the previous error \"stdin: is not a tty\" -- fixing this now...') || exit 0;"

        # Run Any Extra Config
        device.vm.provision :shell , path: "./helper_scripts/config_switch.sh"

        # Apply the interface re-map
        device.vm.provision "file", source: "./helper_scripts/apply_udev.py", destination: "/home/vagrant/apply_udev.py"
        device.vm.provision :shell , inline: "chmod 755 /home/vagrant/apply_udev.py"
        device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -a A0:00:00:00:00:16 eth0"
        device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -vm "
        device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -s"
        device.vm.provision :shell , :inline => $script

  end





  config.vm.define "leaf07" do |device|
    device.vm.provider "virtualbox" do |v|
      v.name = "leaf07"
      v.memory = 512
    end
    device.vm.hostname = "leaf07"
    device.vm.box = "cumulus-vx-3.0.0"


        device.vm.network "private_network", virtualbox__intnet: "net_l7_eth0", auto_config: false , :mac => "A00000000017"

        device.vm.network "private_network", virtualbox__intnet: "net_#{wbid}_21", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_#{wbid}_23", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_#{wbid}_31", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_#{wbid}_31", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_#{wbid}_37", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_#{wbid}_37", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l7s1", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l7s2", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l7s3", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l7s4", auto_config: false


    # Disabling the default synced folder
    device.vm.synced_folder ".", "/vagrant", disabled: true
#    device.vm.provision :shell , path: "./helper_scripts/pre_config.sh"
#    device.vm.provision "file", source: "./helper_scripts/rename_eth_swp", destination: "/home/vagrant/rename_eth_swp"
#    device.vm.provision "file", source: "./helper_scripts/autogenerated/leaf07_remap_eth", destination: "/home/vagrant/remap_eth"
#    device.vm.provision :shell , inline: "mv /home/vagrant/rename_eth_swp /etc/init.d/rename_eth_swp"
#    device.vm.provision :shell , inline: "mv /home/vagrant/remap_eth /etc/default/remap_eth"
#    device.vm.provision :shell , inline: "chmod 755 /etc/init.d/rename_eth_swp"
#    device.vm.provision :shell , inline: "/etc/init.d/rename_eth_swp verbose"

    device.vm.provider "virtualbox" do |vbox|
        vbox.customize ['modifyvm', :id, '--nicpromisc2', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc3', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc4', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc5', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc6', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc7', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc8', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc9', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc10', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc11', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc12', 'allow-vms']

        vbox.customize ["modifyvm", :id, "--nictype1", "virtio"]

    end

        # Fixes "stdin: is not a tty" message --> https://github.com/mitchellh/vagrant/issues/1673
        device.vm.provision :shell , inline: "(grep -q -E '^mesg n$' /root/.profile && sed -i 's/^mesg n$/tty -s \\&\\& mesg n/g' /root/.profile && echo 'Ignore the previous error \"stdin: is not a tty\" -- fixing this now...') || exit 0;"

        # Run Any Extra Config
        device.vm.provision :shell , path: "./helper_scripts/config_switch.sh"

        # Apply the interface re-map
        device.vm.provision "file", source: "./helper_scripts/apply_udev.py", destination: "/home/vagrant/apply_udev.py"
        device.vm.provision :shell , inline: "chmod 755 /home/vagrant/apply_udev.py"
        device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -a A0:00:00:00:00:17 eth0"
        device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -vm "
        device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -s"
        device.vm.provision :shell , :inline => $script

  end





  config.vm.define "leaf08" do |device|
    device.vm.provider "virtualbox" do |v|
      v.name = "leaf08"
      v.memory = 512
    end
    device.vm.hostname = "leaf08"
    device.vm.box = "cumulus-vx-3.0.0"


        device.vm.network "private_network", virtualbox__intnet: "net_l8_eth0", auto_config: false , :mac => "A00000000018"

        device.vm.network "private_network", virtualbox__intnet: "net_#{wbid}_21", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_#{wbid}_23", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_#{wbid}_31", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_#{wbid}_31", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_#{wbid}_37", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_#{wbid}_37", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l8s1", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l8s2", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l8s3", auto_config: false

        device.vm.network "private_network", virtualbox__intnet: "net_l8s4", auto_config: false


    # Disabling the default synced folder
    device.vm.synced_folder ".", "/vagrant", disabled: true
#    device.vm.provision :shell , path: "./helper_scripts/pre_config.sh"
#    device.vm.provision "file", source: "./helper_scripts/rename_eth_swp", destination: "/home/vagrant/rename_eth_swp"
#    device.vm.provision "file", source: "./helper_scripts/autogenerated/leaf08_remap_eth", destination: "/home/vagrant/remap_eth"
#    device.vm.provision :shell , inline: "mv /home/vagrant/rename_eth_swp /etc/init.d/rename_eth_swp"
#    device.vm.provision :shell , inline: "mv /home/vagrant/remap_eth /etc/default/remap_eth"
#    device.vm.provision :shell , inline: "chmod 755 /etc/init.d/rename_eth_swp"
#    device.vm.provision :shell , inline: "/etc/init.d/rename_eth_swp verbose"

    device.vm.provider "virtualbox" do |vbox|
        vbox.customize ['modifyvm', :id, '--nicpromisc2', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc3', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc4', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc5', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc6', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc7', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc8', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc9', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc10', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc11', 'allow-vms']
        vbox.customize ['modifyvm', :id, '--nicpromisc12', 'allow-vms']

        vbox.customize ["modifyvm", :id, "--nictype1", "virtio"]

    end

        # Fixes "stdin: is not a tty" message --> https://github.com/mitchellh/vagrant/issues/1673
        device.vm.provision :shell , inline: "(grep -q -E '^mesg n$' /root/.profile && sed -i 's/^mesg n$/tty -s \\&\\& mesg n/g' /root/.profile && echo 'Ignore the previous error \"stdin: is not a tty\" -- fixing this now...') || exit 0;"

        # Run Any Extra Config
        device.vm.provision :shell , path: "./helper_scripts/config_switch.sh"

        # Apply the interface re-map
        device.vm.provision "file", source: "./helper_scripts/apply_udev.py", destination: "/home/vagrant/apply_udev.py"
        device.vm.provision :shell , inline: "chmod 755 /home/vagrant/apply_udev.py"
        device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -a A0:00:00:00:00:18 eth0"
        device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -vm "
        device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -s"
        device.vm.provision :shell , :inline => $script

    end
end
