# Vim & NeoVim

## Install

```bash
sudo pacman -S vim 
sudo pacman -S neovim
```

## config

```
ln -s  ~/.vim .local/share/nvim/site
mkdir .config/nvim
ln ~/.vimrc .config/nvim/init.vim
```

### check healthy

* exec : `:checkhealth`

#### python3 provider

```
yay -S python-pip
pip install pynvim
```

#### python2 provider

```
yay -S python2
yay -S python2-pip
pip2 install pynvim
```

#### node provider

```
yay -S node
yay -S npm
sudo npm install -g neovim
```

#### ruby provider

```
yay -S ruby
gem install neovim
gem environment
```



### use plugin manager

#### install

```bash
curl -fLo ~/.vim/autoload/plug.vim --create-dirs \
    https://raw.githubusercontent.com/junegunn/vim-plug/master/plug.vim
```

### Theme

* morhetz/gruvbox
* connorholyday/vim-snazzy

#### more usuage see [Vim-Plug](https://github.com/junegunn/vim-plug)

## usuage

#### Opreation Motion

a provider which transparently uses shell commands to communicate with the

* opreation

  ```
  d   Cut
  y	Copy
  p	Paste
  c	delet and enter write mode
  
  ```

* motion

  ```
  f find
  i in
  w word
  ```

  