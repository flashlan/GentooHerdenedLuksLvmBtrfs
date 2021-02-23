# GentooHerdenedLuksLvmBtrfs
Instalação Gentoo hardened com incriptação Luks, Lvm e Btrfs em subvolumes


# Instalação Gentoo: Hardened, Lvm, Encriptação Luks e Btrfs Subvolumes
Criado segunda 22 fevereiro 2021

Gentoo Hardened oferece multiplos serviços adicionais de segurança, oferecendo a ativação de várias opções de mitigação de riscos como: Pax, SELinux, GrSecurity e mais....

LVM (Logical Volume Manager)  permite administradores criarem meta dispositivos que proveem uma camada de abstração entre o sistema de arquivos e os armazenamento físico onde á usado. 

Encriptaremos a unidade central, e por sua vez todas as subunidades LVM  e  subvolumes btrfs também estarão encriptadas (menos a partição boor e efi)

![](file:///home/orfeu/Pictures/Diagrama-hds.png)



Assumindo que que a instalação será feita a partir de um ambiente linux:

Download minimal cd install:
<https://www.gentoo.org/downloads/>

Grave  a imagem iso para o pendrive:
	root #dd if=install-amd64-minimal-20210221T214504Z.iso of=/dev/sdb bs=4M


Após dar boot com o live-usb e entrar no terminal use blkid para pegar o nome e UUIDs das unidades
	blkid

(sda = unidade alvo de instalação no nosso caso)

``Assumindo que não será instalado em dual-boot!``

### Particionamento

Vamos criar uma unidade efi de 100Mb, uma segunda unidade para os arquivos de boot (não criptografados) e o restante dos disco criaremos uma unidade lvm para o restante das partições:
OBS:eu deixo 1,5gb para poder usar vários kernels, 256MiB mínimo
	parted -a optimal /dev/sda
	mklabel gpt
	mkpart primary 1MiB 100MiB
	name 1 efi
	set 1 esp on
	mkpart primary 100MiB 1600MiB
	name 2 boot
	set 2 boot on
	mkpart primary 1600MiB 100% 
	name 3 data-encrypted
	set 3 lvm on
	print
	quit 


Critografar nossa unidade:
	cryptsetup -s 512 --use-random luksFormat /dev/sda3


Abriremos ela em seguida:
	cryptsetup luksOpen /dev/sda3 data-decrypted


Com unidade cryptografada aberta, criaremos, respectivamente, os grupos físicos (Physical Groups), grupos de volumes (Volume-Groups)  e os volumes lógicos (logical Volumes):
	pvcreate /dev/mapper/data-decrypted
	vgcreate datavg /dev/mapper/data-decrypted
	lvcreate -L 15GB  -n swap datavg
	lvcreate -l 100%VG -n root datavg


Formatar as partições:
	mkfs.fat -F 32 /dev/sda1
	mkfs.ext4 -L boot /dev/sda2
	mkfs.btrfs -L root /dev/mapper/datavg-root


Criar sub volumes btrfs para conter a rais do sistema "/" no interior da pasta rootfs e  a pasta "/home", ambos como subdiretóios da pasta "@", além de uma pasta snapshots, a qual virá a ser utilizada posteriormente para realização de backups:
	mount /dev/mapper/datavg-root /mnt/gentoo
	mkdir /@
	btrfs subvolume create /@/rootfs
	btrfs subvolume create /@/home
	btrfs subvolume create /snapshots
	umount /mnt/gentoo


### Chroot

Montar o sistema. criar swap e entrar na raiz:
	mount /dev/mapper/datavg-root -o subvol=/@/rootfs /mnt/gentoo
	mkswap /dev/mapper/datavg-swap
	swapon  /dev/mapper/datavg-swap
	cd /mnt/gentoo


Sincronize o relógio:
	ntpd -qg


Instalação stage3
	curl --list-only ftp://gentoo.osuosl.org/pub/gentoo/releases/amd64/autobuilds/current-stage3-amd64-hardened

	wget ftp://gentoo.osuosl.org/pub/gentoo/releases/amd64/autobuilds/current-stage3-amd64-hardened/current-stage3-amd64-hardened-20210221T214504Z.tar.xz

	tar xpvf sstage3-amd64-hardened-20210221T214504Z.tar.xz --xattrs-include='*.*' --numeric-order

	mirrorselect -i -o >> /mnt/gentoo/etc/portage/make.conf
	cp -L /etc/resolv.conf /mnt/gentoo/etc


Montar o Sistema para chroot
	mount --types proc /proc /mnt/gentoo/proc
	mount --rbind /sys /mnt/gentoo/sys
	mount --make-rslave /mnt/gentoo/sys
	mount --rbind /dev /mnt/gentoo/dev
	mount --make-rslave /mnt/gentoo/dev
	chroot /mnt/gentoo /bin/bash


``chroot``
	source /etc/profile
	export PS1="(chroot) $PS1"


Montar as partições restantes
	mkdir -f /home
	mkdir /boot
	mount /dev/sda2 /boot
	mount /dev/mapper/datavg-root subvol=/@/home /home
	mkdir -p /boot/efi
	mount /dev/sda1 /boot/efi


Povoar o sistema de arquivos
	emerge-webrsync
	emerge --sync
	eselect profile list
	eselect profile <num> # selecione hardened
	$EDITOR /etc/portage/make.conf


**``USE_FLAGS=``**
Básicas e recomendadas:"bash-completion hardened -jit vim-syntax xattr"
Desktop:"alsa dbus X pulseaudio xft" 
**``COMMON_FLAGS=``"-02 -march=native -mtune=native -pipe"**
**``MAKEOPTS=``"-jN"** 
(Onde "N" é o número de cores do seu processador)

Após configurar o sistema:
	emerge -avuDN @world


Fuso-horário
	echo "America/Sao_Paulo" > /etc/timezone
	emerge --config sys-libs/timezone-data
	$EDITOR /etc/locale.gen 

	pt_BR UTF-8
	C.UTF8 UTF-8
	locale-gen
	locale -a
	eslect locale list
	eselect locale set <num>
	env-update
	source /etc/profile
	export PS1="(chroot) $PS1"


Hostname
	$EDITOR /etc/conf.d/hostname
	$EDITOR /etc/hosts


Definir a senha root
	passwd 


Adicionar um usuário (grupos podem ser wheel, kvm, etc..)
	useradd -g users -G <group1>,<group2> -m <nome de usuário> 
	passwd <nome de usuário>


	echo 'L10N = "en-US pt-BR"' > /etc/portage/make.conf
	$EDITOR /etc/conf.d/keymaps

*keymap="br-abnt2"*

	$EDITOR /etc/conf.d/hwclock

*clock="UTC"*

	
	$EDITOR /etc/fstab #substitua UUIDs usando blkid

Modelo de fstab com todas as unidades finais (ps10 é para usar chave de descriptografia no pendrive e não precisar digitar senha toda vez que liga, use tmpfs caso use ssd ou nvme, para montar a partição /tmp na memoria ram, aumentando assim a vida útil do disco, pela questão da messiva escrita de dados que ocoree nessa partição, a pasta [/data](file:///home/orfeu/git/MyTiddly/ZimGentoo/data) esta localizada em outro hdd, servindo de armazenamento exclusivo para receber os backups da pasta snapshots).
	UUID=01b9e6e2...         /boot        ext4    noauto,noatime                 		     1 2
	UUID=01b9e6e2...         /            btrfs   subvol=/@/rootfs,ssd,noatime   		     0 1
	/dev/mapper/datavg-root  /home        btrfs   subvol=/@/home,ssd,noatime     		     0 2
	/dev/mapper/datavg-swap  none         swap    sw                             		     0 0
	UUID=cfb6e4fb-...        /data        btrfs   noauto,defaults               		     0 2
	UUID=775d7d14-...        /media/ps10  ext4    defaults,nofail                  		     0 2
	/dev/mapper/datavg-root  /mnt/raiz    btrfs   noauto,ssd,noatime			 		     0 2
	tmpfs                    /tmp         tmpfs   rw,nosuid,noatime,nodev,size=4G,mode=1777  0 0
	#/dev/cdrom              /mnt/cdrom   auto   noauto,ro                                   0 0

	emerge -av app-portage/gentoolkit
	emerge -av app-admin/doas # substituto para o sudo
	emerge -av app-admin/metalog #logs do sistema
	rc-update add metalog  default 
	emerge -av sys-process/cronie
	rc-update add cronie default
	emerge -av net-misc/dhcpd


##### Configurar a rede
Caso o método escolhido seja por ifrc, mais simples e usando cabo de rede apenas:
***OBS: ***
***touch /etc/udev/rules.d/80-net-name-slot.rules***
***para renomear sua interfaces de redes para eth0, wlan0, etc...)***
	emerge -av net-misc/netifrc
	echo 'config_eth0="dhcp"' >> /etc/conf.d/net
	ln -s /etc/init.d/net.lo /etc/init.d/net.eth0
	rc-update add net.eth0 default


TODO: network manager

##### Opcional
Caso opte por interface gráfica:
	emerge -av x11-wm/openbox
	emerge -av x11-base/xorg-server
	emerge -av x11-misc/tint2
	echo 'tint2 &' > ~/.xinitrc
	echo 'exec dbus-launch --exit-with-session  openbox-session' > ~/.xinitrc


Utilitários
	emerge -av vim tmux mc



##### Compile Kernel
Baixe o códifo fonte e adicione algumas use flags necessárias (euse)
	emerge -av sys-kernel/gentoo-sources sys-kernel/linux-firmware
	euse -p sys-apps/util-linux -E static-libs
	euse -p sys-kernel/genkernel -E  cryptsetup
	emerge -av -av sys-kernel/genkernel
	emerge -av sys-fs/btrfs-progs sys-fs/dosfstools
	$EDITOR /etc/genkernel.conf 
	etc-update


*MENUCONFIG="yes"*
*LVM="yes"*
*LUKS="yes"*
*BTRFS="yes"*
*FIRMWARE="yes"*
*FIRMWARE_SRC="/lib/firmware"*

	genkernel all

Ative suporte no kernel marcando com "*" os seguintes itens:

* ☑ Device Drivers
	* ☑ Multiple device Drivers support (RAID and LVM)
		* ☑ Device Mapper support
		* ☑ Crypt Target Support
* ☑ Cryptographic API
	* ☑ XTS Support
	* ☑ SHA224 and SHA256 digest algorhitm
	* ☑ AES cypher algorithm (x86_64)
	* ☑ User-space interface for hash algorithms
	* ☑ User-space interface for simmmetric keys and algorithms


##### Grub
	ls /boot/
	echo 'GRUB_PLATFORMS="efi-64"'>> /etc/portage/make.conf
	euse -p sys-boot/grub -E mount device-mapper
	emerge -av sys-boot/os-prober
	emerge -av sys-boot/grub:2 
	mount -o remount,rw /sys/firmware/efi/efivars
	grub-install --target=x76_64-efi --efi-directory=/boot/efi
	$EDITOR /etc/default/grub

insert "dolvm dobtrfs root=UUID=<datavg-root-uuid>" em GRUB_CMD_LINE_DEFAULT=""
~~(na minha configuração tambem precisei configurar "root=" como "real_root="  após  o grub gerar o grub.cfg)~~
	grub-mkconfig -o /boot/grub/grub/cfg


Finalizar
	exit
	cd 
	umount -l /mn/tgentoo/dev{/shm,/pts,}
	umount -R /mnt/gentoo
	reboot








