解压
cp /boot/initrd.img-5.10.110-rockchip-rk3588 ./

@#mv initrd.img-5.10.110-rockchip-rk3588 initrd.img-5.10.110-rockchip-rk3588.gz

@#gunzip initrd.img-5.10.110-rockchip-rk3588.gz

cpio -i -d < initrd.img-5.10.110-rockchip-rk3588

@#rm -rf initrd.img-5.10.110-rockchip-rk3588


打包

@#find . -print |cpio -o -H newc > ../initrd.img-5.10.110-rockchip-rk3588

@#gzip -9 initrd.img-5.10.110-rockchip-rk3588

mv initrd.img-5.10.110-rockchip-rk3588.gz initrd.img-5.10.110-rockchip-rk3588

mkimage -A arm -O linux -T ramdisk -C gzip -d initrd.img-5.10.110-rockchip-rk3588 uInitrd


镜像优化


@#e2label /dev/mapper/loop0p2 opi_root

parted test.img -s -- mkpart primary ext4 513M 3.5G

parted test.img -s -- mkpart primary ext4 3.51G 4.5G

映射loop设备
sudo losetup -f --show {镜像文件}
/dev/loop0

device mapper
sudo kpartx -va /dev/loop0
add map loop0p1 (254:0): 0 257 linear /dev/loop0 256
add map loop0p2 (254:1): 0 18015 linear /dev/loop0 513

格式化
sudo mkfs.vfat /dev/mapper/loop0p1
......
sudo mkfs.ext4 /dev/mapper/loop0p2
......

