# Ubuntu on Ubuntu by libvirt
In this repo, I recorded (and later verified it by reproducing the steps) the steps involed when I created a Ubuntu 20.04 VM (Virtual Machine) with a shared folder with the host machine (in my case is Ubuntu 18.04).

Note: this tutorial can be executed fully through a remote machine with a VNC client (I used RealVNC on macOS) except the step when you need to enable the Virtualization depending on your CPU vendor (say Intel or AMD).

The core features you will have after walking through this tutorial:

* Static IP by DHCP mac address binding so that you can easily `ssh` into it.
* A shared folder between the host and the guest machine.
* A guest machine that can be accessed by a VNC client (like `VNC Viewer`).

## Environment
* Host machine information:
    * `lsb_release -a`:
        ```
        No LSB modules are available.
        Distributor ID:	Ubuntu
        Description:	Ubuntu 18.04.5 LTS
        Release:	18.04
        Codename:	bionic
        ```
    * `uname -a`:
        ```
        Linux anji-MS-7B89 5.4.0-65-generic #73~18.04.1-Ubuntu SMP Tue Jan 19 09:02:24 UTC 2021 x86_64 x86_64 x86_64 GNU/Linux`
        ```

## Steps
* Enable your system VM acceleration technology. In my case, I enabled `SVM Mode`:

    ![amd](./amd-vm-enable.jpg)

* Install required packages:

    ```
    $ sudo apt update
    $ sudo apt install qemu-kvm libvirt-daemon-system
    $ sudo apt install virt-manager
    ```

* Add your user to the `libvirt` group in order to manage it through the current user:
    ```
    $ sudo adduser $USER libvirt
    ```

* Download the Ubuntu 20.04 ISO image from <https://releases.ubuntu.com/20.04/>

* Install the guest machine by `virt-install` (You can tune the below options to meet your needs):

    ```
    $ vm_name="ubuntu-virtinstall"
    $ disk_path="/var/lib/libvirt/images/ubuntu-virtinstall.qcow2"
    $ img_path="your ubuntu 20.04 path"
    $ virt-install --virt-type kvm --name "$vm_name"  --ram 2048 --disk "$disk_path",format=qcow2,size=20 --network  network=default --graphics vnc,listen=127.0.0.1 --noautoconsole --os-type=linux --os-variant=ubuntu18.04 --cdrom="$img_path"
    ```

    * NOTE: the image at first should be owned by you. But because of `dynamic_ownership = 1` (See `/etc/libvirt/qemu.conf` for more info), you image will later be owned by `libvirt-qemu` and the group of `kvm`. Like this (after the ste:

        ```
        $ ls -al ubuntu-20.04.1-desktop-amd64.iso
        -rw-rw-r-- 1 libvirt-qemu kvm 2785017856  八   1  2020 ubuntu-20.04.1-desktop-amd64.iso
        ```

* Get your guest machine's VNC display port and use a VNC client (say RealVNC to connect it and finish the installation process):

    ```
    $ virsh vncdisplay ubuntu-virtinstall
    127.0.0.1:0
    ```

    * NOTE: You will need to turn the `Picture Quality` of the VNC client to `High`. Or it will keep showing failure...

* Wait for your guest machine to finish installation! And then `virsh start ubuntu-virtinstall`

* Let your virtual machine to have a fixed IP address by (See <https://libvirt.org/formatnetwork.html> for more information).
    ```
    $ virsh  dumpxml ubuntu-virtinstall | grep 'mac address'
          <mac address='52:54:00:b2:58:de'/>
    $ virsh net-edit default
    ...
    ...
    ...

    $ virsh net-destroy default
    $ virsh net-start default
    ```
    * My `default` network's xml looks like this:
        ```xml
        <network>
          <name>default</name>
          <uuid>53e1adb3-6d58-4be7-95a0-e489d89f1539</uuid>
          <forward mode='nat'>
            <nat>
              <port start='1024' end='65535'/>
            </nat>
          </forward>
          <bridge name='virbr0' stp='on' delay='0'/>
          <mac address='52:54:00:ab:ce:77'/>
          <ip address='192.168.122.1' netmask='255.255.255.0'>
            <dhcp>
              <range start='192.168.122.2' end='192.168.122.254'/>
              <host mac='52:54:00:b2:58:de' name='ubuntu-virtinstall' ip='192.168.122.3'/>
            </dhcp>
          </ip>
        </network>
        ```

* Remember to run `sudo apt install -y openssh-server` in the guest machine. So that you can `ssh` into it!

* Create a directory with mode `777` (because we will need to give permission for `libvirt-qemu` to write things to your folder.
    ```
    $ cd ~
    $ mkdir ubuntu-virt-shared
    $ chmod 777 ubuntu-virt-shared
    ```

    * NOTE: If you think `chmod 777` is too dirty. You might see other solutions like using `setfacl -m u:libvirt-qemu:rwx ubuntu-virt-shared`. It will NOT work because in the guest machine, the folder may look like this `drwxrwxr-x  2        1001        1001 4096  二  21 20:25 .`. So even as guest user account, you will be treated as `other` which is given `r-x` permission.

* Paste this section into `<devices> ... </devices>` by `virsh edit ubuntu-virtinstall`:

    ```
    <filesystem type='mount' accessmode='mapped'>
      <driver type='path'/>
      <source dir='/home/YOUR USER NAME!/ubuntu-virt-shared/'/>
      <target dir='label'/>
    </filesystem>
    ```

    * NOTE: I think the most confusing part is `<target dir='label'/>`. The value in `dir` does not indicate the real path. It is JUST a label!

* `ssh` to your guest machine and then:
    ```
    $ cd ~
    $ mkdir shared
    $ sudo mount label ./shared -t 9p -o trans=virtio
    $ cd shared
    $ touch test
    $ ls -al
    total 12
    drwxrwxrwx  2        1001        1001 4096  二  20 23:50 .
    drwxr-xr-x 16 ubuntu-virt ubuntu-virt 4096  二  20 23:47 ..
    -rw-rw-r--  1 ubuntu-virt ubuntu-virt    0  二  20 23:50 test
    ```

    * NOTE: I set my guest machine's user as `ubuntu-virt`. This may depends on your setting.

* Back into your host, you will find a file created by `libvirt-qemu`. Because we use accessmode - `mapped`:
    ```
    $ cd ~/ubuntu-virt-shared
    $ ls -al
    ...
    -rw-------  1 libvirt-qemu kvm           0  二  20 23:50 test
    ```

* In order to make the shared folder permanent, you need to add this line into your `/etc/fstab` in your guest machine.
    ```
    label    /home/ubuntu-virt/shared/      9p  trans=virtio,rw    0   0
    ```
* Add these lines inside `/etc/initramfs-tools/modules` and then update the initramfs by `sudo update-initramfs -u`:
    ```
    9p
    9pnet
    9pnet_virtio
    ```

* That's all! You've successfully set up a virtual machine!

## FAQ
* Why does my VNC client (VNC Viewer by RealVNC) show two cursors (one with small dot and one with a normal cursor)?
    * After my tweaking, this is caused by the default video card `cirrus`. By deleting the `<video>...</video>` section and adding these lines into `virsh edit ubuntu-virtinstall`, two cursors issue will disappear. It seems to be `cirrus` problem:
        ```
        <video>
          <model type='qxl' ram='65536' vram='65536' vgamem='16384' heads='1' primary='yes'/>
        </video>
        ```

        * NOTE: Or I think you can add `--video qxl` when you run `virt-install`.
        * Mine: `cirrus` section looks like this:
            ```
            <video>
              <model type='cirrus' vram='16384' heads='1' primary='yes'/>
              <alias name='video0'/>
              <address type='pci' domain='0x0000' bus='0x00' slot='0x02' function='0x0'/>
            </video>
            ```

## References
* [https://forums.tomshardware.com/threads/what-is-svm-mode-in-bios.3415554/](https://forums.tomshardware.com/threads/what-is-svm-mode-in-bios.3415554/)
* <https://ubuntu.com/server/docs/virtualization-libvirt>
* [https://gowa.club/Linux-Unix/RealVNC的VNC-Viewer连接到VM虚拟机出错的问题.html](https://gowa.club/Linux-Unix/RealVNC%E7%9A%84VNC-Viewer%E8%BF%9E%E6%8E%A5%E5%88%B0VM%E8%99%9A%E6%8B%9F%E6%9C%BA%E5%87%BA%E9%94%99%E7%9A%84%E9%97%AE%E9%A2%98.html)
* <https://libvirt.org/formatnetwork.html>
* <https://superuser.com/questions/341594/qemu-vnc-how-to-use-absolute-pointing-device>
* <https://serverfault.com/questions/627238/kvm-libvirt-how-to-configure-static-guest-ip-addresses-on-the-virtualisation-ho>
* <https://bugs.launchpad.net/ubuntu/+source/libvirt/+bug/1784001>: Libvirt `dynamic_ownership`'s behavior
* <https://lists.gnu.org/archive/html/qemu-devel/2010-05/msg02673.html>: `passthrough` vs `mapped`
* <https://thoughts.jonathantey.com/how-to-set-up-passthrough-shared-folder-on-kvm/>
* <https://www.linux-kvm.org/page/9p_virtio>
* <https://codingbee.net/rhcsa/rhcsa-the-acls-mask-setting>: how to use `setfacl`. But it turns out not wokring.
