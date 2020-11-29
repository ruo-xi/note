# Custom desktop enviroment

## xorg config

```bash
yay -S xorg-xinitrc

## double card  eg:intel and nvidia
yay -S optimus-manager
```

#### xrandr 

​	 It can be used to set the size, orientation or reflection of the outputs for a screen. 

```bash
ex:
xrandr --output HDMI-1-2 --size 3840x2160 --rate 60
--off 
```



## sound

```bash
# sound server
# backend
yay -S pulseaudio
yay -S pulseaudio-alsa
# frontend
yay -S pacmixer


yay -S alsa-utils
```



## DeskTop Environmeny

### disaplay manager

#### lightdm

### windows manager

#### dwm

```bash
# fix java appication see https://wiki.archlinux.org/index.php/Java#Gray_window,_applications_not_resizing_with_WM,_menus_immediately_closing

# .xinrtrc
export _JAVA_AWT_WM_NONREPARENTING=1
```



### terminal emulator

#### st

### file manager

#### ranger

### text editor

#### vim

#### neovim

### application launcher

#### dmenu

### Audio volume control

#### alsamixer

```
sudo pacman -S alsa-utiles
```

### Clipboard manager

### Desktop Compositor

#### picom

```
picom --experimental-backends
```



### Desktop wallpaper setter

#### feh

```
sudo pacman -S feh

nvim .xinitrc
# add:
# feh --bg-scale --randomize ~/store/picture/wallpaper/* &
```



### Display power saving settings

### Logout dialogue

### Mount tool

### Notification daemon

### Polkit authentication agent

### Screen locker

## terminal

### st(simple terminal)

## Font

* [ttf-symbola](https://aur.archlinux.org/packages/ttf-symbola/)
* wqy-microhei

```
# sudo pacman -S xorg-xlsfonts

# tty

```

## sound

```
yay -S alsa-uils
# include alsamixer 
```

