# RPis as nodes for a k8s cluster

## Pick the OS and flash on a SDCard
Keep a page open to this link:

https://learnaddict.com/2016/02/23/modifying-a-raspberry-pi-raspbian-image-on-linux/

Download the RPi os image. I pick the 2020-12-02-raspios-buster-armhf-lite.zip from RaspberryPi site.
Unzip it.
Do an `fdisk` in order to collect some infos about image:
```
fpezzati@oem-OMEN-by-HP-Laptop-15-ce0xx tmp $ sudo fdisk -l 2020-12-02-raspios-buster-armhf-lite.img
Disk 2020-12-02-raspios-buster-armhf-lite.img: 1,7 GiB, 1858076672 bytes, 3629056 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x067e19d7

Device                                    Boot  Start     End Sectors  Size Id Type
2020-12-02-raspios-buster-armhf-lite.img1        8192  532479  524288  256M  c W95 FAT32 (LBA)
2020-12-02-raspios-buster-armhf-lite.img2      532480 3629055 3096576  1,5G 83 Linux
```
Make a couple of dir where to mount the two images:
```
sudo mkdir -p /mnt/image/boot
sudo mkdir -p /mnt/image/root
```
Compute offsets for boot and root images:
- 8192 * 512 = 4194304 for boot
- 532480 * 512 = 272629760 for root

Mount images by using offsets obtained by previous `fdisk`:
```
sudo mount -v -o offset=4194304 -t vfat 2020-12-02-raspios-buster-armhf-lite.img /mnt/image/boot
sudo mount -v -o offset=272629760 -t ext4 2020-12-02-raspios-buster-armhf-lite.img /mnt/image/root
```
Ok. From now on I will be able to change the raspios image to my needs. Great. Now, what my needs are?
Well, I have a few problems:
- no internet for my cluster's nodes,
- no micro-HDMI cable to get a screen on my RPis.

RPi raspios (raspbian too?) has a built in user, pi, with password raspberry. I have to change that later.

I need a dhcp server and some ssh keys to share with my PRis to reach them from my laptop. I can use my router to provide ip address to my RPis, for each one I have to:
- tell my RPi to start the ssh server,
- provide a hostname,
- change RPi pi user's password (default is raspberry).
No need to provide a ssh public key, I'll do after install.

### Tell my RPi to start the ssh server
Having previously mounted the boot partition I have to enable ssh server on the RPi, to do so I simply have to put a file called `ssh` file in `/mnt/image/boot`. Should be sufficient.
```
sudo touch /mnt/image/boot/ssh
```
I also create a .ssh folder in pi's home:
```
sudo mkdir /mnt/image/root/home/pi/.ssh
```
and a public-private key pair with ssh-keygen with no passphrase (maybe to use one could be a good idea but I am lazy):
```
ssh-keygen -t rsa -b 2048 -f rpi_id_rsa
```
I'll use that one later.

### Provide a hostname
To change the hostname editing or substituting the RPi /etc/hostname file should be sufficient. Default is `raspberrypi`. You have to be root to achieve this.
```
echo "nestor1" > /mnt/image/root/etc/hostname
```
to name my RPi as nestor1. This will allow me to distinguish it between all the RPis.

### Change RPi pi user's password
Well, there is no real need to change it now. I'll change it when OS is installed.

Finally, accordingly to the blog where I read this, I have (not really mandatory) to reduce the amount of memory dedicated to the GPU by adding a line to /mnt/image/boot/config.txt:
```
echo "gpu_mem=16" >> /mnt/image/boot/config.txt
```
My RPis won't do anything graphical. They neither will do anything with wifi or IOT for now, so no need to configure 8192CU (wifi) or IC2 driver and modules (interact with breadboard I guess) stuff.

That should be all.

Then let's write the image on the sd card. Plug the micro sd card into the adapter, check the adapter switch and put it into read and write mode if necessary. Plug the adapter into the laptop ad see which device hosts the sd card:
```
sudo fdisk -l
```
check the right device: mine was `Disk /dev/mmcblk0: 59,5 GiB, 63864569856 bytes, 124735488 sectors`, a 64GB sd card. Last step: copy the image file into the card:
```
sudo dd bs=4M if=2020-12-02-raspios-buster-armhf-lite.img of=/dev/mmcblk0 conv=fsync
```
Now my first RPi should be ready. I power it and plug into my network. I expect to see a `nestor1` hosted into my network. No it does not, something went wrong.

**BEWARE!**
Always check the sdCard adapter switch. In my case I wasted a lot of time trying to flash sdCards that were in RO mode because the switch went from RW to RO mode while plugging it in my laptop.

To have the three RPis is sufficient to pick the customized image, mount it again as before and insert another hostname in `/mnt/image/root/etc/hostname`. I added "nestor2" and "nestor3". Ok now I have all my three RPis. Let's use them as k8s nodes.

My bad, I forgot to change the `/etc/hosts` of each of my RPi, I have to remove `raspberry` entry and add a `nestor1` (or `nestor2` or `nestor3`) for each of my RPis.

## The provisioning part
Now that my RPis are up and running, I have to provide them some software, installing k8s is my main aim.

Raspberry Pi OS does need no cgroup `configuration` so I can go straight to ansible:
```
sudo apt update
sudo apt install software-properties-common
sudo apt-add-repository ppa:ansible/ansible
sudo apt install ansible
```
and you get ansible.

## Trying Ansible
Generate a RSA key pair (already done) and copy the .pub one on each RPi:
```
scp /home/fpezzati/.ssh/id_rsa.pub pi@nestor1:~/.ssh/authorized_keys
scp /home/fpezzati/.ssh/id_rsa.pub pi@nestor2:~/.ssh/authorized_keys
scp /home/fpezzati/.ssh/id_rsa.pub pi@nestor3:~/.ssh/authorized_keys
```
oh crap, permission denied error.. I have to add `x` permission on .ssh folder of each RPi first..
```
chmod u+x .ssh
```
then I copy the key using `ssh-copy-id` command. This is not enought... I have to:
```
sudo chown pi .ssh/authorized_keys && chmod 644 .ssh/authorized_keys && chmod 700 .ssh
```
on each RPi, and then I can login passwordless. Now I am ready to build my first inventory.

Inventory made, I ran my first ansible command and I get an error:
```
fpezzati@oem-OMEN-by-HP-Laptop-15-ce0xx ansible $ ansible -i ./hosts all ping
Traceback (most recent call last):
  File "/usr/bin/ansible", line 44, in <module>
    import ansible.constants as C
ImportError: No module named 'ansible'
```
Removed by `apt remove ansible` and installed again by `python -m pip install --user ansible` which put ansible under the right python (I have two, 2.7 and 3.5. Ansible went under 2.7 but I am using 3.5 of course). Now it works.

I have to specify the `pi` user to make ansible works:
```
ansible -i ./hosts all -m ping --user pi
```
that was my 'ansible is working' test.

Do you want to turn off all your RPis at once? Run this:
```
ansible -i ./hosts all -m command -a "sudo shutdown" --user pi
```
Ok hello-world done. Let's put down some playbook.

Modules I met so far:
- gather_facts: fetches a lot of infos about selected hosts,
- script: executes a sh or bash command specified as argument (using -a),
- ping: check connection between selected hosts,
- command: ?
- apt: it is like doing apt on every selected hosts.

Having an ansible.cfg file where you run ansible let you override the main ansible's config.

Run a module as root on the selected hosts by using `become` and `ask-become-pass`, for example:
```
ansible rpi.k8s.cluster -m apt -a update_cache=true --become --ask-become-pass
ansible rpi.k8s.cluster -m apt -a upgrade=dist --become --ask-become-pass
```
it will asks for sudo password.

### Playbooks
A playbook is a .yaml file like this:
```
-
  name: somename
  hosts: host or a group
  tasks:
    - name:
      module we want to use (e.g.: command)
    - name:
      script: runthis.sh
-
  name: another play name
  hosts:
  tasks:
    - name:
      command:
...
```
.yaml file is a collection of plays. Each play has a name, a hosts as target, a list of tasks.

The `hosts` property indicates a host or group registered in the inventory file.

Because they are lists order matters, order of occurrence in list is the order of execution.

The `command` property indicates which ansible module to run.

To run a playbook use `ansible-playbook` tool:
```
ansible-playbook playbook_of_choice --user pi
```

Ansible provides a `when` module to do conditional stuff in playbooks (I don't think it is a good idea). However ansible provides some variables keeping info about hosts, for example: `ansible-distribution`. A good point to get which variables are available is by getting `gather_facts` module output.

You can provide a list instead of a single entry to the `command` entry in your playbook:
```
---

- hosts: all
  tasks:
  - name: do more than one thing
    apt:
      upgrade: dist
      name:
      - nginx
      - git
      state: latest
```

To refer to variables in playbooks you can use `"{{ variable_name }}"` and declare those in inventory file bounding them to a host or group:
```
nestor1 my_favorite_web_server=nginx my_favourite_version_tool=git
```
well.. you are hardcoding variables here...

You can specify a group of your inventory file as `hosts` value.

Ok, my playbook is complete. It's a trivial one, I'll try to translate into roles for the future. For now it is ok.

Ok now I have kubeadm on each RPi. Is this what I need?
My laptop will be my master node and I already have kubectl installed on.

Looks I have to create a CA certificate and sign it with a key to provide a Certificate Authority for the cluster... I thought it was simpler. For now I am following this guide:

https://github.com/mmumshad/kubernetes-the-hard-way


## Notes
Caste told me not to go this way but go for a classical install in a RPi V.3 and, once done, get a backup image from that to install i my other RPis (v.4). Use the RPi v.3 to get a backup image because it has a HDMI port, so I can plug a screen on.
