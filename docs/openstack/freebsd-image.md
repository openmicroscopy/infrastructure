# FreeBSD OpenStack image creation

See [Remy van Elst's
guide](https://raymii.org/s/tutorials/FreeBSD_11.0-release_Openstack_Image.html)
to creating images, which this document is based upon, with some
modifications.  The instructions are updated for new releases, so it
would be worth looking at these when updating these instructions for a
new release where some of the details may have changed.

## Local setup

Create python virtualenv for openstack modules

```sh
if [ -f ~/venv/openstack/bin/activate ]; then
    source ~/venv/openstack/bin/activate
else
    mkdir -p ~/venv
    virtualenv ~/venv/openstack
    source ~/venv/openstack/bin/activate

    pip install python-openstackclient
fi

source ~/.openstackrc
```

Set additional details required below:

```sh
ssh_key="your_ssh_key"
security_group="required_security_group"
```

## ISO image creation

Download ISO image, then run:

```sh
openstack image create --file ~/Downloads/FreeBSD-11.0-RELEASE-amd64-disc1.iso --disk-format iso --container-format bare "FreeBSD-11.0-RELEASE-amd64-dvd1.iso"
```

Set `image_id` using `id` of created image.

## Installation

```sh
openstack volume create --size 16 "FreeBSD-11.0-root"
```

Set `volume_id` using `id` of created volume.

Next, boot the installer:

```sh
openstack server create "--image=$image_id" --key-name "$ssh_key" --flavor "m1.small" --block-device-mapping "vdb=$volume_id:::0" "FreeBSD-11.0-RELEASE-install"
```

Set `install_vm_id` using `id` of created VM.

Now go to the OpenStack console for this VM, and run through the
installation sequence:

- Perform standard FreeBSD install
- Keyboard:            UK keymap
- Hostname:            freebsd110
- Distribution select: No package sets (keep minimal)
- Partitioning:        Auto (UFS)
- Partition:           Entire disk
- Partition scheme:    GPT
- Root password:       freebsd110 (temporary)
- Network configuration: vtnet0, IPv4 DHCP, IPv6 SLAAC, default resolver setup
- Clock:                 UTC
- Timezone:              UTC
- System configuration:  sshd, dumpdev
- System hardening:    Stack guard, Clean /tmp, Disable remote syslog, Disable sendmail
- Add user accounts:     None
- Open shell
  - add `ifconfig vtnet0 mtu 1450` to `/etc/rc.local` to work around [#187094](https://bugs.freebsd.org/bugzilla/show_bug.cgi?id=187094)
  - run `shutdown -p now`

Now stop and remove the installation VM, retaining the volume:

```sh
# Seems pointless if already stopped; can be skipped
openstack server stop "$install_vm_id"

openstack server remove volume "$install_vm_id" "$volume_id"

openstack server delete "$install_vm_id"
```

## Post-installation setup of cloudinit

Make the volume bootable:

```sh
cinder set-bootable "$volume_id" true
```

List available networks:

```sh
openstack network list
```

Set `net_id` using `id` for required network (must permit outgoing
external connections for downloading packages).

Now, start up a new VM using our installation volume:

```sh
openstack server create --volume "$volume_id" --block-device "source=volume,id=$volume_id,dest=volume,shutdown=preserve,bootindex=0" --flavor "m1.small" --security-group "$security_group" --nic "net-id=$net_id" --key-name "$ssh_key" FreeBSD-11.0-RELEASE-configure
```

Set `configure_vm_id` using `id` of created VM.

Log into VM as root using the OpenStack console.  Set up a minimal
Python installation and configure booting:


```sh
pkg install vim-lite py27-setuptools ca_root_nss
touch /firstboot
rehash
easy_install eventlet
easy_install iso8601
cat <<EOF >>/boot/loader.conf
console="comconsole,vidconsole"
autoboot_delay="15"
EOF
```

Next, install cloudinit itself:

```sh
fetch https://raw.github.com/pellaeon/bsd-cloudinit-installer/master/installer.sh
chmod +x installer.sh
./installer.sh
```

Finally, tidy up:

```sh
set history = 0
history -c
dd if=/dev/zero of=/bla
rm /bla
shutdown -p now
```

Remove the configuration VM:

```sh
openstack server delete "$configure_vm_id"
```

## Create OpenStack image

Finally, create an OpenStack image from this volume:

openstack image create --volume "$volume_id" "FreeBSD-11.0-RELEASE"

This will be used to create new VMs.

## Test

Create a VM using our new image:

```sh
openstack server create --image="FreeBSD-11.0-RELEASE" --security-group "$security_group" --nic "net-id=$net_id" --key-name "$ssh_key" --flavor "m1.small"  "test7"
```

You should be able to log in and do whatever you like.