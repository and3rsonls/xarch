<p align="center"><img src="https://i.imgur.com/UjnRFbR.png" width="500"/></p>

# Minha Instalação ArchLinux com GPT EFI LUKS e Bootctl
<a id="^top"></a>
### PREPARANDO A ISO
> - link para download do archLinux iso e pgp<br>
[https://www.archlinux.org/download/](https://www.archlinux.org/download/)

```
$ gpg --keyserver-options auto-key-retrieve --verify archlinux-versao-x86_64.iso.sig
    - verificando assinatura da ISO
```
#### OU
```
$ pacman-key -v archlinux-versão-x86_64.iso.sig
	- se você estiver em um archlinux
```

```
# dd bs=4M if=archlinux-versão-x86_64.iso of=/dev/sdX status=progress && sync
    - gravando a iso em um pendrive
```

### INICIALIZANDO A LIVE ISO
```
# wifi-menu
	- para conexão wireless
```

```
# ping -c3 ping archlinux.org
	- testando a conexão
```

```
# loadkeys br-abnt2
	- teclado
```

```
# modprobe -a dm-mod dm-crypt
	- carregar módulos
```

### CRIANDO AS PARTIÇÕES

```
# lsblk
	- identificando o disco
```

```
# gdisk /dev/sda
	- criar uma nova tabela de partição vazia tipo GPT
```
####  layout das partições necessárias

| nome | tamanho | tipo | montagem |
| :--: | :-----: | :--: | :------: |
| sda1 | 512M    | ef00 - efi| /boot    |
| sda2 | ***M    | 8e00 - lvm| /mnt    |

### _CRIPTOGRAFANDO_<sup>[1](#1)</sup> A PARTIÇÃO LINUX LVM

```
# cryptsetup -c aes-xts-plain64 -y -s 512 luksFormat /dev/sda2
	- criptografando a partição "/dev/sda2"
```

```
# cryptsetup luksOpen /dev/sda2 aux
	- entrando na partição criptografada
```

```
# pvcreate /dev/mapper/aux
	- criando o physical volume (pv)
```

```
# pvs
	- informações sobre o pv
```

```
# vgcreate arch /dev/mapper/aux
	- criando o volume group (vg)
	- vgcreate <nome do grupo> <caminho do physical volume>
```

```
# lvcreate -L 3g arch -n swap
# lvcreate -l 100%FREE arch -n root
	- criando as logical volume (lv)
	- lvcreate -L <tamanho |M|G> <nome do grupo> -n <nome do logical volume>
```

```
# lvs
	- informações sobre o logical volume (lv)
```

### FORMATANDO E MONTADO AS PARTIÇÕES

```
# mkfs.ext4 /dev/mapper/arch-root
# mount /dev/mapper/arch-root /mnt
	- formatando e montando a partição root
```

```
# mkswap /dev/mapper/arch-swap
# swapon /dev/mapper/arch-swap
	- formatando e montando a partição swap
```

```
# mkfs.fat -F32 /dev/sda1
	- formatado a partição boot
```

```
# mkdir /mnt/{boot,home}
	- criando as partições boot e home em '/mnt'
```

```
# mount /dev/sda1 /mnt/boot
	- montando a partição boot
```

### INSTALAÇÃO INICIAL

```
# pacstrap /mnt base base-devel linux-lts linux-lts-heades linux-firmware lvm2 vim
	- instalando o sistema base
```

```
# genfstab -U /mnt > /mnt/etc/fstab
	- gerando a tabela de partições
```

```
# arch-chroot /mnt
	- entrando no ambiente chroot
```

```
# echo "KEYMAP=br-abnt2" > /etc/vconsole.conf
	- configurando o teclado
```

```
# passwd root
	- senha para o usuário root
```

```
# useradd -m -G wheel -s /bin/bash archer
# passwd archer
	- adicionar um novo usuário e uma senha
```

```
# vim /etc/sudoers
	- adicionar o usuário 'archlinux' ao sudoers
```

### INITRAMFS

```
# vim /etc/mkinitcpio.conf
	- procurar e editar as seguintes linhas seguindo o exemplo abaixo
```
> .<br>
> .<br>
> .<br>
> MODULES=(***ext4***)<br>
> .<br>
> .<br>
> .<br>
> HOOKS=(...modconf block ***keymap encrypt lvm2 resume*** filesystems ...)<br>
> .<br>
> .<br>
> .

```
# mkinitcpio -p linux
	- recriar o initramfs
```

### _BOOTCTL_<sup>[2](#2)</sup>

```
# booctl --path=/boot$esp install
	- instalar o bootctl
```

```
# blkid -o export /dev/sda2 | grep -w UUID > /boot/loader/entries/arch.conf
	- criar o arquivo "arch.conf"
```

```
# vim /boot/loader/entries/arch.conf
	- editar o arquivo criado anteriormente conforme o exemplo abaixo
```
> title archlinux<br>
> linux /vmlinuz-linux<br>
> initrd /initramfs-linux.img<br>
> options cryptdevice=UUID=478d1dd9-39ab-4fde-b2b2-f60b6aa0752a:root resume=/dev/mapper/arch-swap root=/dev/mapper/arch-root rw<br>
> <br>
> **"resume"** é responsável pela hibernação 

```
# vim /boot/loader/loader.conf
	- editar o arquivo conforme o exemplo abaixo
```
> default arch<br>
> timeout 3<br>
> console-mode max<br>
> editor no

```
# booctl update
	- atualizar os arquivos de boot
```

### CONEXÃO DE REDE
```
# pacman -S iw wireless_tools network-manager-applet dhcpcd networkmanager
```

### REINICIAR SISTEMA

```
# exit
# umount -R /mnt && swapoff /dev/sdX
# reboot
```

### TUDO CERTO?
_sobreviveu ao primeiro boot?_<br>
 login<br>
 passwd<br>
	- login root

### CONEXÃO

#### conexão cabeada

```
# systemctl enable NetworkManager.service
# systemctl start NetworkManager.service
	- habilitar e iniciar o networkmanager
```

```
# ping -c3 archlinux.org
	- checar conexão
```

#### conexão wi-fi
#### layout usando o wpa_supplicant

| aplicativo     | status | systemd |
| :------------: | :----: | :-----: |
| NetworkManager | iniciado   | ativado |
| wpa_supplicant | iniciado  | ativado  |
| dhcpcd         | parado   | desativado |

```
# ip a
	- verificar o nome da interface
```

```
# systemctl enable wpa_supplicant
# systemctl start wpa_supplicant
	- habilitar e iniciar o wpa_supplicant
```

```
# wpa_passphrase ESSID PASSWD > /etc/wap_supplicant/wpa_supplicant-wlp3s0.conf
	- gerar o arquivo de configuração da interface wireless
```

```
# wpa_supplicant -B -iwlp3s0 -c/etc/wap_supplicant/wpa_supplicant-wlp3s0.conf
	- habilitar a configuração do "wpa_supplicant"
```

```
# dhcpcd wlp3s0
	- concluir a conexão usando o dhcpcd
```

```
# ping -c3 archlinux.org
	- checar conexão
```


### MAIS CONFIGURAÇÕES

```
# vim /etc/pacman.conf
	- descomentar para adicionar o repositório multilib
```
> .<br>
> .<br>
> .<br>
> [multilib]<br>
> Include = /etc/pacman.d/mirrorlist<br>
> .<br>
> .<br>
> .

```
# echo "archerhost" > /etc/hostname
	- hostname
```

```
# echo "LANG=pt_BR.UTF-8" > /etc/locale.conf
# export LANG=pt_BR.UTF-8
	- idioma do sistema
```

```
# rm -f /etc/localtime
# ln -s /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime
	- localidade do sistema
```

```
# hwclock --systohc --utc
	- ajuste da hora no /etc/adjtime
```

### AMBIENTE _DESKTOP_<sup>[3](#3)</sup>

**_XORG_**<sup>[4](#4)</sup>
```
# pacman -S xorg xf86-input-synaptics xf86-video-ati
	- o básico para ter um desktop AMD
```

**_XFCE4_**<sup>[5](#5)</sup>
```
# pacman -S xfce4 xfce4-goodies xdg-user-dirs slim archlinux-themes-slim
	- instalando o xfce como desktop
```

**_AUR_**<sup>[6](#6)</sup> **_TRIZEN_**<sup>[7](#7)</sup>
```
$ git clone https://aur.archlinux.org/trizen.git
	- fazer o download do trizen para administração de pacotes AUR
```

```
$ cd trizen
$ makepkg -si
	- para instalar o Trizen
```

**SEGURANÇA**
```
#pacman -S nftables clamav seahorse kkleopatra rkhunter
```

**UTILITÁRIOS**
```
# pacman -S opencl-mesa acpid acpi llvm numlockx ethtool dialog gparted gpart redshift exfat-utils reiserfsprogs nilfs-utils f2fs-tools xfsprogs jfsutils ntfs-3g mtools polkit iputils gvfs ntp wol psutils t1utils usbutils
```

**VIRTUALIZAÇÃO**
```
pacman -S wine
```

**COMPARTILHAMENTO DE ARQUIVOS**
```
# pacman -S transmission-gtk filezilla netcat wget git
```

**NAVEGADORES**
```
# pacman -S chromium firefox-i18n-pt-br links lynx

$ trizen -S tor-browser
    - gerenciador de arquivos AUR
```

**EDITORES DE TEXTO**
```
# pacman -S leafpad mousepad vim
```

**TTF**
```
# pacman -S font-bh-ttf noto-fonts noto-fonts-extra sdl2_ttf ttf-bitstream-vera ttf-caladea ttf-carlito ttf-croscore ttf-dejavu ttf-hack ttf-junicode ttf-linux-libertine perl-font-ttf ttf-anonymous-pro ttf-cormorant ttf-droid ttf-fantasque-sans-mono ttf-fira-code ttf-fira-mono ttf-fira-sans ttf-ibm-plex ttf-inconsolata ttf-indic-otf ttf-ionicons ttf-jetbrains-mono ttf-joypixels ttf-linux-libertine-g ttf-nerd-fonts-symbols-mono ttf-opensans ttf-proggy-clean ttf-roboto ttf-roboto-mono ttf-ubuntu-font-family 

```

**GERENCIADORES DE ARQUIVOS**
```
# pacman -S trash-cli catfish rclone rsync meld 
```

**ARQUIVAMENTO**
```
# pacman -S xarchiver-gtk2 p7zip unzip unrar zip
```

**MULTIMÍDIA**
```
# pacman -S ffmpeg schroedinger libtheora libvorbis libmpeg2 xine-lib libde265 xvidcore gst-libav inkscape wavpack jasper a52dec libmad libvpx geeqie libdca dav1d libdv faad2 x265 x264 faac aom flac lame libxv opus gimp

$ trizen -S codecs64
    - gerenciador de pacotes AUR
```

**AUDIO**
```
# pacman -S pulseaudio-alsa pavucontrol alsa-utils alsa-oss alsa-lib
```

**VÍDEO**
```
# pacman -S vlc
```

**OFFICE**
```
# pacman -S libreoffice-still libreoffice-still-pt-br galculator-gtk2 retext xsane cups
```

**CLIENTES EMAIL**
```
# pacman -S claws-mail mutt
```

**LEITORES**
```
# pacman -S calibre mcomix xpdf
```

**WEBCAM**
```
# pacman -S cheese
```

**IRC**
```
# pacman -S hexchat
```

**ÁREA DE TRABALHO REMOTA**
```
# pacman -S x11vnc

$ trizen -S real-vnc-viewer
    - gerenciador de pacotes AUR
```

**DESENVOLVIMENTO**
```
# pacman -S tk tcl
```

**MONITORES DE SISTEMA**
```
# pacman -S conky
```

**INFORMAÇÕES DO SISTEMA**
```
# pacman -S hwdetect neofetch hwinfo htop
```

**LOG**
```
# pacman -S pacmanlogviewer
```

**LIMPEZA**
```
# pacman -S bleachbit
```

**AGENDADORES**
```
# pacman -S cronie
```

**PERNONALIZAÇÃO**
```
# pacman -S numix-themes-archblue capitaine-cursors xcursor-neutral papirus-icon-theme
```

**GAMES**
```
# pacman -S dwarffortress asciiportal stone-soup

```

## Leitura complementar<br>

<a id="1" href="#"><sup>1</sup></a>
[LUKS e LVM](https://williamcanin.me/blog/instalando-archlinux-com-criptografia-luks-e-lvm/)<br>
[LUKS](https://www.redhat.com/sysadmin/disk-encryption-luks)<br>
[DOCUMENTAÇÃO](https://github.com/libyal/libluksde/tree/master/documentation)<br>
[DM-CRYPT](https://wiki.archlinux.org/index.php/Dm-crypt/Device_encryption)<br>

<a id="2" href="#"><sup>2</sup></a>
[SYSTEMBOOT](https://wiki.archlinux.org/index.php/systemd-boot)<br>
[MANUAL](https://www.freedesktop.org/software/systemd/man/bootctl.html)<br>
[BOOTCTL](http://man7.org/linux/man-pages/man1/bootctl.1.html)

<a id="3" href="#"><sup>3</sup></a>
[DESKTOP](https://wiki.archlinux.org/index.php/Desktop_environment_(Portugu%C3%AAs))

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

