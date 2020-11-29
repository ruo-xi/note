# SoftWare

## terminal

* yakuake

## shell

* zsh + oh-my-zsh

  ```
  sudo pacman -S zsh
  
  #安装 oh-my-zsh
  sh -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)" 
  #一般会报错 因为raw.githubusercontent.com 被墙了 但是我翻墙也不能访问 可以通过hosts文件解决 
  #在文尾添加 199.232.4.133 raw.githubusercontent.com 
  #配置方法 省略（～～）
  ```

  

* fish

  ```
  sudo pacman -S fish
  #配置
  fish_config
  #禁用greeting
  set fish_greeting ""
  ```

## Git

```
yay -S openssh
ssh-keygen -t rsa -C "your_email@example.com"'
ssh -T git@github.com

git config --global user.name "ruo-xi"
git config --global user.email "314386327@qq.com"

```



## aur package manager

* yay

  ```bash
  git clone https://aur.archlinux.org/yay.git
  cd yay
  makepkg -si # 期间go安装依赖时 遇到443 我的解决方法是使用代理
  #设置代理
  go env -w GOPROXY="https://mirrors.aliyun.com/goproxy"
  
  
  #设置yay代理
  code .bashrc
  #在文件中添加
  #proxy(){
  #    export http=http://ProxyHost:Port
  #   export https=http://ProxyHost:Port
  #}
  #在bash环境中使用proxy函数使用代理 只在当前终端中生效 
  ```

## List of application

```
# ide
idea
goland
webstorm
clion
vscode
vim

# shot
flameshot

# windows manager
dolphin

# markdown editor
typora

# pdf reader
foxitreader

#archive and compress
zip  unzip
tar
rar unrar

#container
docker

#browser
chromium

#go out to foreign 
qv2ray
v2raya
```

