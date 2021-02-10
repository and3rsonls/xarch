# <p align="center"><img src="https://i.imgur.com/UjnRFbR.png" width="500"/></p>

# Minha Instalação de uso pessoal do ArchLinux com LUKS e Grub
<a id="^top"></a>
## PREPARANDO A ISO
> - link para download do archLinux iso e pgp<br>
[https://www.archlinux.org/download/](https://www.archlinux.org/download/)

~~~
$ gpg --keyserver-options auto-key-retrieve --verify archlinux-versao-x86_64.iso.sig
	- verificar assinatura da ISO

$ pacman-key -v archlinux-versão-x86_64.iso.sig
	- verificar em um archlinux

# dd bs=4M if=archlinux-versão-x86_64.iso of=/dev/sdX status=progress && sync
	- gravar a ISO em um pendrive
~~~

## INICIALIZAR A LIVE ISO
~~~
# loadkeys br-abnt2
	- layout do teclado
~~~

## REALIZANDO CONEXÃO

#### wi-fi
~~~
# iwctl
[iwd]# help
[iwd]# device list
[iwd]# station wlan0 get-networks
[iwd]# station wlan0 connect SSID
	- para conexão wireless

# ping archlinux.org
	- testar a conexão
~~~

#### ethernet
~~~
# systemctl status systemd-networkd.service
	- para verificar conexão wired

# ping archlinux.org
	- testar a conexão
~~~

### CRIANDO AS PARTIÇÕES
~~~
# lsblk
	- identificar o disco

# cfdisk /dev/sda
	- criar uma nova tabela de partições vazia tipo DOS
	- marcar '/dev/sda1' como bootável 
~~~
####  layout das partições necessárias

| nome | tamanho | tipo | montagem |
| :--: | :-----: | :--: | :------: |
| sda1 | 512MiB  | linux | /mnt/boot    |
| sda2 | *GiB    | linux | /mnt    |

### _CRIPTOGRAFANDO_<sup>[1](#1)</sup> A PARTIÇÃO LINUX LVM
~~~
# modprobe -a dm-mod dm-crypt
	- carregar módulos para criptografia

# cryptsetup -c aes-xts-plain64 -y -s 512 luksFormat /dev/sda2
	- criptografar a partição '/dev/sda2'

# cryptsetup luksOpen /dev/sda2 aux
	- entrar na partição criptografada

# pvcreate /dev/mapper/aux
	- criar o physical volume (pv)

# pvs
# pvdisplay
	- informações sobre o pv

# vgcreate arch /dev/mapper/aux
	- criar o volume group (vg)
	- vgcreate <nome do grupo> <caminho do physical volume>

# lvcreate -L 3g arch -n swap
# lvcreate -l 100%FREE arch -n root
	- criar as logical volume (lv)
	- lvcreate -L <tamanho |M|G> <nome do grupo> -n <nome do logical volume>

# lvs
# pvdisplay
	- informações sobre o logical volume (lv)
~~~

### FORMATANDO E MONTADO AS PARTIÇÕES
~~~
# mkfs.ext4 /dev/mapper/arch-root
# mount /dev/mapper/arch-root /mnt
	- formatar e montar a partição root

# mkdir /mnt/{boot,home}
	- criar os diretórios boot e home em '/mnt'

# mkswap /dev/mapper/arch-swap && swapon /dev/mapper/arch-swap
	- formatar e montar a partição swap

# mkfs.ext4 /dev/sda1
	- formatar a partição boot

# mount /dev/sda1 /mnt/boot
	- montar a partição boot
	
# lsblk
	- listando tipos de partições
	
# fdisk -l /dev/sdX
	- listando partições
~~~

### INSTALAÇÃO INICIAL
~~~
# pacstrap /mnt base base-devel linux-lts linux-lts-headers linux-firmware grub lvm2 vim
	- instalar o sistema base

# genfstab -U /mnt > /mnt/etc/fstab
	- gerar a tabela de partições

# arch-chroot /mnt
	- entrar no ambiente chroot

# echo "KEYMAP=br-abnt2" > /etc/vconsole.conf
	- configurar o teclado

# passwd root
	- senha para o usuário root

# useradd -m -G wheel -s /bin/bash archer
# passwd archer
	- adicionar um novo usuário e uma senha

# vim /etc/sudoers
	- adicionar o usuário 'archer' ao sudoers
~~~

### CONFIGURAÇÃO DE REDE
~~~
# echo "archerhost" > /etc/hostname
	- configuração do hostname

# vim /etc/hosts
	- criar configuração do hosts

-----------------------------------------
127.0.0.1     localhost
1::           localhost
127.0.3.1     archerhost.local archerhost
-----------------------------------------
    - adicionar ao arquivo '/etc/hosts'
~~~
### CONEXÃO DE REDE
~~~
# pacman -S networkmanager wireless_tools nm-connection-editor network-manager-applet dhcpcd wpa_supplicant net-tools
	- aplicativos necessários para conexão ethernet ou wi-fi
~~~

#### wi-fi
~~~
# systemctl disable dhcpcd.service
	- desativar-lo pois ele pode causar instabilidade na conexão wi-fi

# systemctl enable NetworkManager.service
	- habilitar o serviço na inicialização

# systemctl enable wpa_supplicant.service
	- habilitar o serviço na inicialização
~~~

#### ethernet
~~~
# systemctl enable dhcpcd.service 
	- habilitar o serviço na inicialização

# systemctl enable NetworkManager.service
	- habilitar o serviço na inicialização

# systemctl disable wpa_supplicant.service
	- desabilitar o serviço na inicialização
~~~

### LOCALIZAÇÃO
~~~
# vim /etc/locale.gen
	- descomentar a localidade para valores monetários regionais, formatos de hora e data, idiossincrasias alfabéticas e outros padrões específicos de localidade.

# locale-gen
	- gerar localidade

# echo "LANG=pt_BR.UTF-8" > /etc/locale.conf
# export LANG=pt_BR.UTF-8
	- idioma do sistema

# rm -f /etc/localtime
# ln -s /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime
	- localidade do sistema

# vim /etc/systemd/timesyncd.conf
	- altere
		NTP=pool.ntp.br
		FallbackNTP=a.ntp.br b.ntp.br c.ntp.br

# hwclock --systohc --utc
	- ajuste o relógio do sistema em '/etc/adjtime'
~~~

### INITRAMFS
~~~
# vim /etc/mkinitcpio.conf
	- procurar e editar as seguintes linhas seguindo o exemplo abaixo

----------------------------------------------------------------------------------
HOOKS=(... modconf block keymap keyboard encrypt lvm2 resume filesystems ...)
----------------------------------------------------------------------------------

# mkinitcpio -p linux-lts
	- recriar o initramfs
~~~

### _GRUB-BIOS_<sup>[2](#2)</sup>
~~~
# vim /etc/default/grub
	- procurar e adicionar a linha abaixo

-------------------------------------------------------------------------------------------------------------------------
GRUB_CMDLINE_LINUX_DEFAULT="cryptdevice=/dev/sda2:aux root=/dev/mapper/arch-root resume=/dev/mapper/arch-swap loglevel=3"
-------------------------------------------------------------------------------------------------------------------------

# grub-install /dev/sda
# grub-mkconfig -o /boot/grub/grub.cfg
	- instalar o grub e carregar as configurações
~~~

### REINICIAR SISTEMA
~~~
# exit
# umount -R /mnt && swapoff /dev/sda2
# reboot
~~~

### PÓS INSTALAÇÃO

### CONEXÃO

#### wi-fi
~~~
$ nmcli device wifi list
	- lista as redes

$ nmcli device wifi connect <SSID> password <SSID_password>
	- conecta a rede

$ nmcli connection show
	- lista as redes conectadas

$ ping archlinux.org
	- checar conexão
~~~

#### ethernet
~~~
$ systemctl status NetworkManager.service
	- verificar conexão

$ ping archlinux.org
	- checar conexão
~~~

### PACMAN.CONF
~~~
# vim /etc/pacman.conf
	- descomentar para adicionar o repositório multilib

-------------------------------------------------------
[multilib]
Include = /etc/pacman.d/mirrorlist
-------------------------------------------------------

# pacman -Syu
	- atualizar o archlinux e o 'multilib'
~~~

### INTERFACE GRÁFICA

**Qual driver de vídeo instalar?**
~~~
# lspci | grep -e VGA -e 3D
~~~

**_XORG_**<sup>[4](#4)</sup>
~~~
# pacman -S xorg xf86-input-synaptics xf86-video-ati
~~~

**_XFCE4_**<sup>[5](#5)</sup>
~~~
# pacman -S xfce4 xfce4-goodies xdg-user-dirs slim archlinux-themes-slim
	- instalando o xfce como desktop
~~~

### SEGURANÇA
~~~
# pacman -S nftables clamav linssid seahorse gpa openssh kleopatra rkhunter
~~~

### UTILITÁRIOS
~~~
# pacman -S nmap lsof opencl-mesa acpid acpi llvm numlockx ethtool dialog gparted gpart redshift exfat-utils reiserfsprogs nilfs-utils f2fs-tools xfsprogs jfsutils ntfs-3g mtools polkit iputils gvfs ntp wol psutils t1utils usbutils baobab zenity
~~~

### VIRTUALIZAÇÃO
~~~
# pacman -S wine
~~~

### COMPARTILHAMENTO DE ARQUIVOS
~~~
# pacman -S transmission-gtk filezilla netcat wget git
~~~

### _AUR_<sup>[6](#6)</sup> _TRIZEN_<sup>[7](#7)</sup>
~~~
$ git clone https://aur.archlinux.org/trizen.git
	- fazer o download do trizen para administração de pacotes AUR

$ cd trizen
$ makepkg -si
	- instalar o Trizen
~~~

### SNAP
~~~
$ git clone https://aur.archlinux.org/snapd.git
	- download do Snap

$ cd snapd
$ makepkg -si
	- instalar o Snap

# systemctl enable --now snapd.socket
	- ativar soquete de comunicação principal do Snap

# ln -s /var/lib/snapd/snap /snap
	- criar link simbólico entre '/var/lib/snapd/snape /snap'
~~~

### NAVEGADORES
~~~
# pacman -S chromium firefox-i18n-pt-br links lynx

$ trizen -S tor-browser
~~~

### EDITORES DE TEXTO
~~~
# pacman -S leafpad mousepad
~~~

### TTF
~~~
# pacman -S xorg-fonts-alias-cyrillic xorg-fonts-alias-misc xorg-fonts-misc xorg-fonts-type1 font-bh-ttf noto-fonts noto-fonts-extra sdl2_ttf ttf-bitstream-vera ttf-caladea ttf-carlito ttf-croscore ttf-dejavu ttf-hack ttf-junicode gnu-free-fonts ttf-linux-libertine perl-font-ttf ttf-anonymous-pro ttf-cormorant ttf-droid ttf-fantasque-sans-mono ttf-fira-code ttf-fira-mono ttf-fira-sans ttf-ibm-plex ttf-inconsolata ttf-indic-otf ttf-ionicons ttf-jetbrains-mono ttf-joypixels ttf-linux-libertine-g ttf-nerd-fonts-symbols-mono ttf-opensans ttf-proggy-clean ttf-roboto ttf-roboto-mono ttf-ubuntu-font-family

$ trizen -S ttf-ms-fonts
~~~

### GERENCIADORES DE ARQUIVOS
~~~
# pacman -S trash-cli catfish rclone rsync meld 
~~~

### ARQUIVAMENTO
~~~
# pacman -S xarchiver-gtk2 p7zip unzip unrar zip
~~~

### MULTIMÍDIA E CODECS
~~~
# pacman -S libdc1394 libdvdcss libgme vcdimager smbclient libnfs protobuf libmicrodns lua-socket live-media libdvdread libdvdnav zvbi libkate libtiger chromaprint lirc projectm libgoom2 ffmpeg schroedinger libtheora libvorbis libmpeg2 xine-lib libde265 xvidcore gst-libav inkscape wavpack jasper a52dec libmad libvpx geeqie libdca dav1d libdv faad2 x265 x264 faac aom flac lame libxv opus gimp

$ trizen -S codecs64
~~~

### AUDIO
~~~
# pacman -S pulseaudio-alsa pavucontrol alsa-utils alsa-oss alsa-lib
~~~

### VÍDEO
~~~
# pacman -S vlc
~~~

### OFFICE
~~~
# pacman -S libreoffice-still libreoffice-still-pt-br galculator-gtk2 retext xsane cups

$ trizen -S pdfsam
~~~

### CLIENTES EMAIL
~~~
# pacman -S claws-mail mutt
~~~

### LEITORES
~~~
# pacman -S calibre mcomix xpdf
~~~

### WEBCAM
~~~
# pacman -S cheese
~~~

### IRC
~~~
# pacman -S hexchat
~~~

### ÁREA DE TRABALHO REMOTA
~~~
# pacman -S x11vnc

$ trizen -S real-vnc-viewer
~~~

### DESENVOLVIMENTO
~~~
# pacman -S tk tcl
~~~

### MONITORES DE SISTEMA
~~~
# pacman -S conky
~~~

### INFORMAÇÕES DO SISTEMA
~~~
# pacman -S hwdetect neofetch hwinfo htop
~~~

### LOG
~~~
# pacman -S pacmanlogviewer
~~~

### LIMPEZA
~~~
# pacman -S bleachbit
~~~

### AGENDADORES
~~~
# pacman -S cronie
~~~

### PERSONALIZAÇÃO
~~~
# pacman -S archlinux-wallpaper capitaine-cursors xcursor-neutral papirus-icon-theme

$ trizen -S numix-themes-archblue
~~~

### GAMES
~~~
# pacman -S dwarffortress asciiportal stone-soup 0ad 0ad-data

$ snap install cncra
    - instalar C&C: Red Alert

$ snap install cnctsun
	- instalar C&C: Red Alert Tiberium Sun

$ snap install cncra2yr
	- instalar C&C: Red Alert 2 Yuri's Revenge

$ snap install the-powder-toy
  	- instalar The Powder Toy
~~~

### LEITURA COMPLEMENTAR<br>

<a id="1" href="#"><sup>1</sup></a>
[DM-CRYPT](https://wiki.archlinux.org/index.php/Dm-crypt_(Portugu%C3%AAs)/Device_encryption_(Portugu%C3%AAs))

<a id="2" href="#"><sup>2</sup></a>
[GRUB](https://wiki.archlinux.org/index.php/GRUB_(Portugu%C3%AAs))

<a id="3" href="#"><sup>3</sup></a>
[INTERFACE GRÁFICA](https://wiki.archlinux.org/index.php/Desktop_environment_(Portugu%C3%AAs))

<a id="4" href="#"><sup>4</sup></a>
[XORG](https://wiki.archlinux.org/index.php/Xorg_(Portugu%C3%AAs))

<a id="5" href="#"><sup>5</sup></a>
[XFCE](https://wiki.archlinux.org/index.php/Xfce_(Portugu%C3%AAs))

<a id="6" href="#"><sup>6</sup></a>
[AUR](https://wiki.archlinux.org/index.php/Arch_User_Repository_(Portugu%C3%AAs))

<a id="7" href="#"><sup>7</sup></a>
[TRIZEN](https://github.com/trizen/trizen)

[^top](#top)

<p align="center"><span style="font-size:10px" >Copyright © Judd Vinet and Aaron Griffin.</span></p>

<p align="center"><span style="font-size:10px" >The Arch Linux name and logo are recognized trademarks. Some rights reserved.</span></p>

<p align="center"><span style="font-size:10px" >Linux® is a registered trademark of Linus Torvalds.</span></p>
