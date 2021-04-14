# Manjaro

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

timedatectl set-local-rtc true

pacman-mirrors -i -c China -m rank
sudo pacman -S yay

yay -S go
go env -w GOPROXY="https://goproxy.io,direct"
go env -w GO111MODULE="auto"

yay -S nodejs npm

yay -S python2
yay -S python2-pip

yay -S v2raya
# port 8888 

chsh yu -s /usr/bin/zsh
sudo ln -sf ~/Github/config/zsh ~/.config/zsh
echo "source ~/.config/zsh/zshrc" > ~/.zshrc
# zsh
# exit
# zsh

yay -S neovim
pip install neovim
pip2 install neovim
sudo npm install neovim
pip install ueberzug
sudo ln -sf ~/Github/config/nvim ~/.config/nvim
# nvim
# proxy
# nvim

# yay -S cxorg-xev

yay -S nvidia

sudo ln -sf ~/Github/config/picom ~/.config/picom
rm -rf ~/.config/picom.conf



yay -S google-chrome
yay -S typora
yay -S rofi
yay -S alacritty
yay -S qv2ray

yay -S fd
yay -S ripgrep

yay -Rns nano


```

### no sound

```bash
aplay -l 
alsamixer
speaker-test
```



Q