install debian 9 stretch on one encrypted btrfs partition including /boot and booting if via EFI


WARNING: The following has got some little mistakes here and there. And not working/TODO is: enter key just once; efi-grub should use grub.cfg file

______________________________________
GET THE OS

Put the debian spin you want (live, incl. nonfree) on a usb stick.

In your UEFI, make sure EFI is being booted, not BIOS-legacy and also switch off secure boot.
Boot live system (e.g. via F12 on power-on), edit "/etc/apt/sources.list", add "contrib non-free"
Make shure to be connected to the internet.
Do "apt-get update ; apt-get install btrfs-tools cryptsetup grub2 grub-efi-amd64-bin"


______________________________________
BOOT PROCESS

EFI-software on mainboard --1--> efi executable on EFI partition (has grub baked in) --2--> grub --3--> initrd/kernel on encrypted partition

you'll have to enter pass after step 2 so grub can access encrypted partition.
when grub passes ctrl to kernel, encrypted device is being unmounted.
so, after step 3 a keyfile will be used by the initrd to access encrypted partition.


______________________________________
PREPARE PARTITIONS

Create 3 Partitions: EFI (512MB), unformatted (8461MB, 8069MiB; will be set up as crypted swap later), root (all the rest of the drive).

At the end of this, root & swap will be encrypted.
Note: Due to the way encryption of swap is set up here, you will loose suspend-to-disk functionality.

Do "cryptsetup benchmark" and take whatever is fastest.
cryptsetup --cipher aes-xts --key-size 256 --hash sha256 --use-random --verify-passphrase luksFormat /dev/sd??
just dont use luks2, since grub can't handle that.

"cryptsetup open /dev/sd?? cryptroot"
"cryptsetup luksDump /dev/sd??" just to visually check.

"mkfs.btrfs /dev/mapper/cryptroot"



______________________________________
THE LAYOUT

now this is supposed to be for a desktop system.

name convention: btrfs subvol-names are prefixed with @

mount /dev/mapper/cryptroot /mnt
btrfs subvolume create /mnt/@               # going to be root
btrfs subvolume create /mnt/@snapshots      # btw, a snapshot is also a subvolume
btrfs subvolume create /mnt/@home-rob
btrfs subvolume create /mnt/@home-rob/@volatile-rob # nested subvol isn't being snapshotted (i.e. snap of @home-rob doesnt contain @volatile-rob)
btrfs subvolume create /mnt/@home-rob/@const-rob

this is what we want in the end:

subvolid=5 /dev/sdX
    @ (mounted as /)
        home
            rob (mounted @home-rob; user specific system settings)
                volatile (mounted @volatile-rob; all user specific documents, sourcecode, everything that's usually edited by the user)
                    docs/work
                    docs/private
                    devel
                const (mounted @const-rob; vm images, movies, everything that should have NO-COW)
                    media (no cow)
                    software-packages (no cow)
                    vm-images (no cow)
                .someCfg dirs, etc. (all the rest usually residing in home/rob)
    .snapshots (mounted @snapshots)

this essentially means, you are able to snapshot independently:
-system/OS stuff ("/")
-home of user rob (user app cfgs, user cfgs, desktop env. configs etc.)
-volatile data for user rob
-const data for user rob


______________________________________
SNAPSHOT EXAMPLES

convention: snapshot names are e.g. "s-2018-01-30-@root", "s-2018-01-30-@home-rob", "s-2018-01-30-@volatile-rob"

Don't do this now, these are just examples on how to take snapshots later, when the system is in use:
btrfs subvolume snapshot -r / /.snapshots/s-2018-02-01-@root
btrfs subvolume snapshot -r /home/rob /.snapshots/s-2018-02-01-@home-rob
btrfs subvolume snapshot -r /home/rob/volatile /.snapshots/s-2018-02-01-@volatile-rob


______________________________________
INSTALL OS

btrfs subvolume list /mnt # find subvol-id of "@"
btrfs subvolume set-default <subvol-id> /mnt
btrfs subvolume get-default /mnt    # just to check

umount /mnt
cryptsetup close cryptroot

now, install OS (i.e. boot debian installer from stick).
decrypt / manually on the installer's cmd-line.
in case of debian 9 stretch non-free + KDE installer, you can get it to load crytpsetup module, by going to the "setup crypto"-menu, but backing out without actually writing anything on disk.
you see short time a info dialog popup containting something like "loading cryptsetup module" or something like that.
then, in the cmd-line you can open the encrypted partition, get partmanager to recognize it, keep whats on there and use it as "/" target for installation.
of course, choose "manual partitioning" and don't use extra /boot partition, just efi, swap and root.
debian installer will fail to install grub bootloader - that's fine for now, just continue to the end of the installation without it.

boot into the live system again and repeat step "GET THE OS" (the apt-get stuff).


______________________________________
PREPARE DIRS

# once in live-system again, mount root and efi partitions
cryptsetup open /dev/sd?? cryptroot" ; mount /dev/mapper/cryptroot /mnt ; mount /dev/sd?? /mnt/boot/efi

# now, the order of operations is important:

cd /home
mv rob rob-old # you can delete it as well

mkdir rob
mount -o compress=lzo,subvol=@home-rob /dev/mapper/cryptroot /mnt/home/rob
mkdir rob/volatile
mkdir rob/const

mount -o compress=lzo,subvol=@volatile-rob /dev/mapper/cryptroot /mnt/home/rob/volatile
mkdir rob/volatile/devel
mkdir rob/docs
mkdir rob/docs/work
mkdir rob/docs/private

mount -o compress=lzo,subvol=@const-rob /dev/mapper/cryptroot /mnt/home/rob/const
mkdir rob/const/media
mkdir rob/const/software-packages
mkdir rob/const/vm-images

chown -R rob:rob rob/

mkdir /.snapshots

# what abou "nodatacow" on a subvolume?
# Within a single file system, it is not possible to mount some subvolumes with nodatacow and others with datacow. The mount option of the first mounted subvolume applies to any other subvolumes.
# So we need to do it for folders.

chattr +C -R /mnt/home/rob/const
lsattr /mnt/home/rob/const # just to check
lsattr /mnt/home/rob/const/vm-images


______________________________________
CHROOT INTO NEW SYSTEM

cd /mnt
mount -o bind /dev /mnt/dev ; mount -o bind /sys /mnt/sys ; mount -t proc /proc /mnt/proc
chroot .


______________________________________
PREPARE INITRD WITH CRYPTO

# not all that is neccessary, but i don't know which one actually takes...
echo 'export CRYPTSETUP=y' >> /etc/initramfs-tools/conf.d/cryptsetup
echo 'export CRYPTSETUP=y' >> /etc/environment
echo 'export CRYPTSETUP=y' >> /etc/initramfs-tools/conf.d/force-cryptsetup
echo 'export CRYPTSETUP=y' >> /usr/share/initramfs-tools/conf-hooks.d/forcecryptsetup
# on debian stretch:
echo 'export CRYPTSETUP=y' >> /etc/cryptsetup-initramfs/conf-hook

blkid|grep LUKS        # get relevant uuid

echo "target=cryptroot,source=UUID=XXX,key=none,rootdev,discard" >> /etc/initramfs-tools/conf.d/cryptroot

dd bs=512 count=4 if=/dev/urandom of=/hd_keyfile.bin     # create keyfile
chmod -R g-rwx,o-rwx /hd_keyfile.bin       # only root may access it

cryptsetup luksAddKey /dev/sd?? keyfile.bin
# enter one already existing pass (i.e. the one, you gave in section PREPARE PARTITIONS)
# partition can now be unlocked with either password or keyfile.

# test keyfile. this prints "0" if keyfile works
cryptsetup --key-file keyfile.bin luksOpen /dev/sd?? someTest --test-passphrase ; echo $?

# preparation for initrd to access hd via keyfile
echo "cryptroot UUID=??? /hd_keyfile.bin luks,keyscript=/bin/cat" >> /etc/crypttab

# put keyfile also into ramdisk via initramfs-tools hook
echo "#!/bin/sh" >> /etc/initramfs-tools/hooks/crypto_keyfile
echo "cp /hd_keyfile.bin \"${DESTDIR}\"" >> /etc/initramfs-tools/hooks/crypto_keyfile
chmod +x /etc/initramfs-tools/hooks/crypto_keyfile

update-initramfs -u -k all      # or -c for create instead of u

lsinitramfs /boot/???   # check if keyfile is in there


______________________________________
PREPARE GRUB WITH CRYPTO

# we want grub to put crypto and btrfs and everything else neccessary into the efi executable

# add "GRUB_ENABLE_CRYPTODISK=y" in /etc/default/grub and
GRUB_CMDLINE_LINUX="cryptdevice=/dev/disk/by-uuid/<UUID_OF_LUKS_DEVICE>:cryptroot cryptkey=rootfs:/<name of keyfile>"
# rootfs:/ means "it's in the initrd". if no cryptkey is specified, this defaults to "crypto_keyfile.bin"
GRUB_CMDLINE_LINUX_DEFAULT="noresume quiet"

grub-mkconfig > /boot/grub/grub.cfg     # isnt used, cfg is baked in .efi executable.. TODO

# use --removable only if you install on external HD
grub-install --efi-directory=/boot/efi --boot-directory=/boot/ --removable --target=x86_64-efi /dev/mapper/cryptroot


______________________________________
PREPARE CRYPTO RELATED STUFF

ln -sf /dev/mapper/cryptroot /dev/cryptroot

ls /dev/disk/by-id/ # find out id of swap partition
echo "swap /dev/disk/by-id/<ID OF SWAP PARTITION> /dev/urandom swap,cipher=aes-cbc-essiv:sha256" >> /etc/crypttab

# edit fstab
/dev/mapper/cryptroot /           btrfs   defaults,compress=lzo,noatime        0       0
/dev/mapper/cryptroot /home/rob       btrfs   defaults,compress=lzo,commit=120,subvol=@home-rob,noatime 0       0
/dev/mapper/cryptroot /home/rob/volatile       btrfs   defaults,compress=lzo,commit=120,subvol=@home-rob/@volatile-rob,noatime 0       0
/dev/mapper/cryptroot /home/rob/const       btrfs   defaults,compress=lzo,commit=120,subvol=@home-rob/@const-rob,noatime 0       0
/dev/mapper/cryptroot /.snapshots       btrfs   defaults,compress=lzo,commit=120,subvol=@snapshots,noatime 0       0
/dev/mapper/swap  none  swap  sw  0   0
                                                               

exit
umount chroot mounts (or don't, doesn't matter much)

# don't do the following, it doesnt do anything, cfg is baked in efi executable.. 
# TODO: make .efi executable user EFI/BOOT/grub.cfg, which should include /boot/grub/grub.cfg we made earlier
cp /mnt/boot/grub/grub.cfg /mnt/boot/efi/EFI/BOOT/grub.cfg 


______________________________________
PREPARE EFI BOOT ENTRY

efibootmgf -v  # see whats there
efibootmgr -b <nr> -B # maybe delete the one which debian installer created.
create new one
efibootmgr -c -d /dev/sd? -p <partitionnr> -L <someLabel> -l /EFI/BOOT/BOOTX64.EFI


______________________________________
DONE

reboot and enjoy
