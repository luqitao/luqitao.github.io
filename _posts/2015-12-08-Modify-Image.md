# Epitome

I want to have os image that can login with password.

download UEC image.
for example, this precise.
http://cloud-images.ubuntu.com/precise/current/precise-server-cloudimg-amd64-disk1.img

this image was forbidden to password login feature in terminal console.
it's reasonable for use of cloud image.

but now, I test sample image for use of OpenStack.
so, I create iamge that can login with password in terminal console.

# Step

- download image  
http://cloud-images.ubuntu.com/precise/current/precise-server-cloudimg-amd64-disk1.img
- mount os image  
this image is qcow2 image. so, use qemu-nbd.  
<pre>
# modprobe nbd
# dmesg | grep nbd
nbd: registered device at major 43
# ls /dev/nbd*
/dev/nbd0  /dev/nbd10  /dev/nbd12  /dev/nbd14  /dev/nbd2  /dev/nbd4  /dev/nbd6  /dev/nbd8
/dev/nbd1  /dev/nbd11  /dev/nbd13  /dev/nbd15  /dev/nbd3  /dev/nbd5  /dev/nbd7  /dev/nbd9
</pre>

<pre>
# qemu-nbd --connect=/dev/nbd0 /home/ubuntu/os_images/precise-server-cloudimg-amd64-disk1.img
# mkdir /mnt/target_vm
# mount /dev/nbd0p1 /mnt/target_vm
</pre>

# chroot to qcow2 image

<pre>
# chroot /mnt/target_vm/
</pre>

# change password for ubuntu user

<pre>
root@ubuntu12:/# passwd ubuntu
Enter new UNIX password:
Retype new UNIX password:
passwd: password updated successfully
root@ubuntu12:/#
</pre>

# terminate chroot

<pre>
# exit
</pre>

# unmount qcow2 image

<pre>
# cd /
# umount /mnt/target_vm
# sudo qemu-nbd --disconnect /dev/nbd0
/dev/nbd0 disconnected
</pre>

# test vm

<pre>
kvm -drive file=./os_images/precise-server-cloudimg-amd64-disk1-custom.img -m 512 -boot d -vnc :0
</pre>

access with vnc, and check that you can login with password!

# URL
http://www.geocities.co.jp/SiliconValley/2994/tool/nvp.html
http://www.postcard.st/nosuz/blog/2011/09/10-14