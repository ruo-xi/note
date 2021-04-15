## Manjaro

```bash
mkdir Github
mkdir .ssh
cd Github
git clone https://github.com/ruo-xi/.config config
mv config/ssh/* ~/.ssh
chmod 400 ~/.ssh/id_rsa
chmod 600 ~/.ssh/id_rsa.pub
rm -rf config 
git clone git@github.com:ruo-xi/scripts
git clone git@github.com:ruo-xi/note
git clone git@github.com:ruo-xi/.config config
ln -s ~/Github/config/Xmodmap/.Xmodmap ~/.Xmodmap
sudo ln -sf ~/Github/config/ranger ~/.config/ranger

# timedatectl set-local-rtc true

sudo pacman-mirrors -i -c China -m rank
sudo pacman -S yay

rm -rf ~/.i3
sudo ln -sf ~/Github/config/i3 ~/.config/i3

sudo ln -sf ~/Github/config/alacritty ~/.config/alacritty
yay -S alacritty
sudo rm -rf /bin/terminal /bin/i3-sensible-terminal



yay -S go
go env -w GOPROXY="https://goproxy.io,direct"
go env -w GO111MODULE="auto"

yay -S nodejs npm

yay -S python2
yay -S python2-pip

# yay -S v2raya
yay -S v2ray-desktop
yay -S qv2ray
# config qv2ray rm v2ray-desktop
# port 8888 

yay -Rns palemoon-bin
yay -S google-chrome

# need proxy
yay -Rns manjaro-zsh-config
chsh yu -s /usr/bin/zsh
sudo ln -sf ~/Github/config/zsh ~/.config/zsh
echo "source ~/.config/zsh/zshrc" > ~/.zshrc
# zsh
# exit
# zsh
# zimfw install

# need proxy
yay -Rns vim
yay -Rns vi
yay -S neovim
pip install neovim
pip2 install neovim
sudo npm install -g neovimk
pip install ueberzug
sudo ln -sf ~/Github/config/nvim ~/.config/nvim
# nvim
# unproxy
# nvim

# yay -S xorg-xev
# setxkbmap -option caps:escape

yay -S nvidia
sudo
sudo ln -sf ~/Github/config/picom ~/.config/picom
rm -rf ~/.config/picom.conf



yay -S typora

yay -S rofi
yay -S fd
yay -S ripgrep

```

Font Configuration

```
yay -S nerd-fonts-source-code-pro
yay -S wqy-microhei
yay -S ttf-symbola
yay -Rns 
ln -sf ~/Github/config/fontconfig ~/.config/fontconfig
fc-cache
```



### no sound

```bash
aplay -l 
alsamixer
speaker-test
```



Q