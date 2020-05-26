# A Linux Development Desktop with VMware Horizon - Part III: Shell


# High :zap: Way to Shell

The operationalization of platforms such as e.g. Kubernetes, or the use of tools for building services or applications such as Docker, requires in both cases the use of the command-line. There are dozens of great plugins, themes and extensions out there to pimp your shell so that it´ll help you to increase velocity as well as useability.

**Part III** of my series is focused on the Shell and which *"Highway"* you can take to have the described tool available at the end.

{{< image src="/img/posts/201912_development_desktop/CapturFiles-20200224_100043.jpg" caption="Figure I: Default Terminal appearance before tuning" src-s="/img/posts/201912_development_desktop/CapturFiles-20200224_100043.jpg" class="center" >}}

## 1. ZSH aka Z shell
http://www.zsh.org/

ZSH is a extended shell with lots of features, support for plugins and themes.

**<span style="color:#018914">Ubuntu:</span>**

```shell
sudo apt install zsh
```

**<span style="color:#6003B6">CentOS:</span>**

Should be installed by default.

```shell
sudo yum install zsh
```

## 2. Oh My ZSH
https://ohmyz.sh/

A community-driven framework for managing your Z shell configuration.

**<span style="color:#018914">Ubuntu:</span>**

```shell
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

**<span style="color:#6003B6">CentOS:</span>**

```shell
sh -c "$(wget -O- https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

Verify if ZSH is set as your new default Shell:

```shell
echo $SHELL
cat /etc/passwd | grep "username"
```

### 2.1 ZSH Syntax Highlighting
https://github.com/zsh-users/zsh-syntax-highlighting

This plugin enables highlighting of commands whilst they are typed and it´ll help you to avoid syntax erros before you run a command.

**<span style="color:#6003B6">CentOS</span>** & **<span style="color:#018914">Ubuntu:</span>**

Let´s clone the repository into our *oh-my-zsh* plugins directory:

```shell
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
```

Enable the plugin in your zsh config file (`~/.zshrc`) in the "plugins section".

```shell
plugins=(git pks zsh-syntax-highlighting)
```

Restart your terminal.

### 2.2 Install Powerline Fonts
https://github.com/powerline/fonts

Needed fonts for the Powerlevel9k theme (step 4.).

**<span style="color:#018914">Ubuntu:</span>**

```shell
sudo apt install fonts-powerline
```

**<span style="color:#6003B6">CentOS:</span>**

```shell
git clone https://github.com/powerline/fonts.git --depth=1
cd fonts
./install.sh
cd ..
rm -rf fonts
```

### 2.3 Powerlevel9k
https://github.com/Powerlevel9k/powerlevel9k

This theme will give your shell a new shine.

**<span style="color:#6003B6">CentOS</span>** & **<span style="color:#018914">Ubuntu:</span>**

```shell
git clone https://github.com/bhilburn/powerlevel9k.git ~/.oh-my-zsh/custom/themes/powerlevel9k
vim ~/.zshrc
```

Replace the default ZSH_THEME with the new Powerlevel9k theme:

```shell
ZSH_THEME="powerlevel9k/powerlevel9k"
```

The Powerlevel9k theme gives you some pretty neat customization options which makes your terminal even more powerful.

Here´s my configuration:

```shell
POWERLEVEL9K_SHORTEN_DIR_LENGTH=3
POWERLEVEL9K_SHORTEN_DELIMITER=””
POWERLEVEL9K_PROMPT_ON_NEWLINE=true
POWERLEVEL9K_SHORTEN_STRATEGY=”truncate_from_right”
POWERLEVEL9K_TIME_BACKGROUND=blue
POWERLEVEL9K_DATE_BACKGROUND=cyan
POWERLEVEL9K_RIGHT_PROMPT_ELEMENTS=(kubecontext time date)
POWERLEVEL9K_LEFT_PROMPT_ELEMENTS=(user dir vcs)
```

And you should also paste the following variable to the top of your *.zshrc* file, otherwise you will get an error when using `tmux` later.

```shell
export TERM="xterm-256color"
```

A logout or reboot of the system is necessary for the changes to take effect.

**The result:**

{{< image src="/img/posts/201912_development_desktop/CapturFiles-20200310_120723.jpg" caption="Figure II: ZSH with Powerlevel-9k Theme | l: Ubuntu r: CentOS*" src-s="/img/posts/201912_development_desktop/CapturFiles-20200310_120723.jpg" class="center" >}}

**I love it!** And it doesn´t only looks good, it also comes in very handy if you work e.g. with `Git` as you can see on *Figure II* for example. The right screenshot shows you that I am in a git branch called *featurebranch* and the color indicates me, that my workingtree is either clean and theres nothing to commit (green) or that new files were added and needs to be commited (orange).

---
**More! More! More!**

## 3 Homebrew on Linux :beer:
https://docs.brew.sh/Homebrew-on-Linux

Homebrew is a package manager (like Snappy) which simplifies the installation of software on your OS and should not be missing if you ask me.

**<span style="color:#6003B6">CentOS</span>** & **<span style="color:#018914">Ubuntu:</span>**

```shell
sh -c "$(curl -fsSL https://raw.githubusercontent.com/Linuxbrew/install/master/install.sh)"
```

The installation-script will throw a **Warning** at the end, because it points out that `/home/linuxbrew/.linuxbrew/bin` isn´t in your PATH.
Add the path to your .zshrc dot-file.

```shell
vim ~/.zshrc
export PATH=$PATH:"/home/linuxbrew/.linuxbrew/bin"
source ~/.zshrc
```

---

## 4 CLI´s and tools
...I´m using and I´d like to recommend

**kubectl** - https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-kubectl-on-linux

{{< admonition info "Info" true >}}
A command line tool for controlling Kubernetes clusters.
{{< /admonition >}}

```shell
brew install kubectl
```

### 4.1 ZSH Autocompletion for `kubectl`
https://v1-16.docs.kubernetes.io/docs/reference/kubectl/cheatsheet/#kubectl-autocomplete

```shell
source <(kubectl completion zsh)
echo "if [ $commands[kubectl] ]; then source <(kubectl completion zsh); fi" >> ~/.zshrc
```

### 4.2 Powershell
https://snapcraft.io/powershell

{{< admonition info "Info" true >}}
PowerShell is an cross-platform (Windows, Linux, and macOS) command-line shell for automation and configuration management.
{{< /admonition >}}

Installation via `snap`:

```shell
sudo snap install powershell --classic
```

### 4.3 KinD - Kubernetes in Docker
https://github.com/kubernetes-sigs/kind

{{< admonition info "Info" true >}}
kind is a tool for running local Kubernetes clusters using Docker container "nodes".
{{< /admonition >}}

{{< image src="/img/posts/201912_development_desktop/CapturFiles-20200227_015150.jpg" caption="Figure III: kind create cluster" src-s="/img/posts/201912_development_desktop/CapturFiles-20200227_015150.jpg" class="center" width="800" height="800" >}}

### 4.4 Octant
https://github.com/vmware-tanzu/octant

{{< admonition info "Info" true >}}
Octant is a tool for developers to understand how applications run on a Kubernetes cluster.
{{< /admonition >}}

![Octant demo](/img/posts/201912_development_desktop/octant-demo.gif)

*<center>Source: https://github.com/vmware-tanzu/octant</center>*

### 4.5 tmux
https://github.com/tmux/tmux

{{< admonition info "Info" true >}}
`tmux` is a terminal multiplexer.
{{< /admonition >}}

**Enable Mouse-scrolling**: Scrolling the terminal pages by using the mouse-wheel is natural for me and because it´s not enabled by default when using `tmux` I need to enable it. Settings like this for example can be applied while running `tmux`. Press *ctrl + b* and then type `:set -g mouse on`.

### 4.6 bat
https://github.com/sharkdp/bat

{{< admonition info "Info" true >}}
`bat` is a cat clone with syntax highlighting and Git integration.
{{< /admonition >}}

### 4.7 glances
https://github.com/nicolargo/glances

{{< admonition info "Info" true >}}
`glances` is a cross-platform monitoring tool.
{{< /admonition >}}

### 4.8 htop
https://github.com/hishamhm/htop

{{< admonition info "Info" true >}}
`htop` is an interactive process viewer.
{{< /admonition >}}

### 4.9 ctop
https://github.com/bcicen/ctop

{{< admonition info "Info" true >}}
`ctop` provides a concise and condensed overview of real-time metrics for multiple containers:
{{< /admonition >}}

### 4.10 prettyping
https://github.com/denilsonsa/prettyping

{{< admonition info "Info" true >}}
`prettyping` is a wrapper around the standard ping tool with the objective of making the output prettier, more colorful, more compact, and easier to read.
{{< /admonition >}}

And all of them can be easily installed via `brew`:
```shell
brew install kind octant tmux bat glances htop ctop prettyping
```

Addtionally to `htop`, `ytop` (former gotop) is a pretty neat process and system monitor.

### 4.11 `ytop`
https://github.com/cjbassi/ytop

```shell
brew tap cjbassi/ytop && brew install ytop
```

{{< image src="/img/posts/201912_development_desktop/CapturFiles-20200217_050729.jpg" caption="Figure IV: tmux, htop, ctop, prettyping & ytop" src-s="/img/posts/201912_development_desktop/CapturFiles-20200217_050729.jpg" class="center" >}}

### 4.12 figlet
http://www.figlet.org/

### 4.13 lolcat
https://github.com/busyloop/lolcat

```shell
brew install figlet lolcat

figlet cliFun | lolcat
```

```shell
      _ _ _____
  ___| (_)  ___|   _ _ __
 / __| | | |_ | | | | '_ \
| (__| | |  _|| |_| | | | |
 \___|_|_|_|   \__,_|_| |_|
```

---

### 4.14 cowsay
You should have a look at this post regarding cowsay :smile:: <a href="https://medium.com/@jasonrigden/cowsay-is-the-most-important-unix-like-command-ever-35abdbc22b7f" target="_blank">"cowsay is the Most Important Unix-like Command Ever"</a>

```shell
brew install cowsay

cowsay -f vader I am your...LOL!

 __________________
< I am your...LOL! >
 ------------------
        \    ,-^-.
         \   !oYo!
          \ /./=\.\______
               ##        )\/\
                ||-----w||
                ||      ||

               Cowth Vader
```

---

***Last but not least!***

### 4.15 Asciinema
https://asciinema.org/

A perfect way how you can easily record your terminal sessions.

```shell
brew install asciinema

asciinema rec
```

<script id="asciicast-aVEG2MvECoVzqtYjkjHWDpw1L" src="https://asciinema.org/a/aVEG2MvECoVzqtYjkjHWDpw1L.js" async></script>

---

## 5 Shell Aliases

Put the following aliases for the individual commands to the end of your `.zshrc` file

```shell
tmux
alias tmuxinit="tmux new-session -n init"
alias c=clear
alias k=kubectl
alias w='watch -n1'
alias ping=prettyping
alias cat=bat
```

# You´re good to go!

**<center>Thanks for reading.</center>**

---

**Change Log:**

- [2020-03-10]: Added Enable Mouse-scrolling for `tmux`; ZSH Syntax Highlighting; Powerlevel9k configuration
- [2020-03-10]: Updated Figure II
- [2020-03-18]: Added Glances & Powershell
