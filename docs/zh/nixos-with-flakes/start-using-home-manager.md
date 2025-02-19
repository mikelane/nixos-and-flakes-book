# 安装使用 Home Manager

前面简单提过，NixOS 自身的配置文件只能管理系统级别的配置，而用户级别的配置则需要使用 home-manager 来管理。

根据官方文档 [Home Manager Manual](https://nix-community.github.io/home-manager/index.html)，要将 home manager 作为 NixOS 模块安装，首先需要创建 `/etc/nixos/home.nix`，配置方法如下：

```nix
{ config, pkgs, ... }:

{
  # 注意修改这里的用户名与用户目录
  home.username = "ryan";
  home.homeDirectory = "/home/ryan";

  # 直接将当前文件夹的配置文件，链接到 Home 目录下的指定位置
  # home.file.".config/i3/wallpaper.jpg".source = ./wallpaper.jpg;

  # 递归将某个文件夹中的文件，链接到 Home 目录下的指定位置
  # home.file.".config/i3/scripts" = {
  #   source = ./scripts;
  #   recursive = true;   # 递归整个文件夹
  #   executable = true;  # 将其中所有文件添加「执行」权限
  # };

  # 直接以 text 的方式，在 nix 配置文件中硬编码文件内容
  # home.file.".xxx".text = ''
  #     xxx
  # '';

  # 设置鼠标指针大小以及字体 DPI（适用于 4K 显示器）
  xresources.properties = {
    "Xcursor.size" = 16;
    "Xft.dpi" = 172;
  };

  # git 相关配置
  programs.git = {
    enable = true;
    userName = "Ryan Yin";
    userEmail = "xiaoyin_c@qq.com";
  };

  # 通过 home.packages 安装一些常用的软件
  # 这些软件将仅在当前用户下可用，不会影响系统级别的配置
  # 建议将所有 GUI 软件，以及与 OS 关系不大的 CLI 软件，都通过 home.packages 安装
  home.packages = with pkgs;[
    # 如下是我常用的一些命令行工具，你可以根据自己的需要进行增删
    neofetch
    nnn # terminal file manager

    # archives
    zip
    xz
    unzip
    p7zip

    # utils
    ripgrep # recursively searches directories for a regex pattern
    jq # A lightweight and flexible command-line JSON processor
    yq-go # yaml processer https://github.com/mikefarah/yq
    exa # A modern replacement for ‘ls’
    fzf # A command-line fuzzy finder

    # networking tools
    mtr # A network diagnostic tool
    iperf3
    dnsutils  # `dig` + `nslookup`
    ldns # replacement of `dig`, it provide the command `drill`
    aria2 # A lightweight multi-protocol & multi-source command-line download utility
    socat # replacement of openbsd-netcat
    nmap # A utility for network discovery and security auditing
    ipcalc  # it is a calculator for the IPv4/v6 addresses

    # misc
    cowsay
    file
    which
    tree
    gnused
    gnutar
    gawk
    zstd
    gnupg

    # nix related
    #
    # it provides the command `nom` works just like `nix`
    # with more details log output
    nix-output-monitor

    # productivity
    hugo # static site generator
    glow # markdown previewer in terminal

    btop  # replacement of htop/nmon
    iotop # io monitoring
    iftop # network monitoring

    # system call monitoring
    strace # system call monitoring
    ltrace # library call monitoring
    lsof # list open files

    # system tools
    sysstat
    lm_sensors # for `sensors` command
    ethtool
    pciutils # lspci
    usbutils # lsusb
  ];

  # 启用 starship，这是一个漂亮的 shell 提示符
  programs.starship = {
    enable = true;
    # 自定义配置
    settings = {
      add_newline = false;
      aws.disabled = true;
      gcloud.disabled = true;
      line_break.disabled = true;
    };
  };

  # alacritty - 一个跨平台终端，带 GPU 加速功能
  programs.alacritty = {
    enable = true;
    # 自定义配置
    settings = {
      env.TERM = "xterm-256color";
      font = {
        size = 12;
        draw_bold_text_with_bright_colors = true;
      };
      scrolling.multiplier = 5;
      selection.save_to_clipboard = true;
    };
  };

  programs.bash = {
    enable = true;
    enableCompletion = true;
    # TODO 在这里添加你的自定义 bashrc 内容
    bashrcExtra = ''
      export PATH="$PATH:$HOME/bin:$HOME/.local/bin:$HOME/go/bin"
    '';

    # TODO 设置一些别名方便使用，你可以根据自己的需要进行增删
    shellAliases = {
      k = "kubectl";
      urldecode = "python3 -c 'import sys, urllib.parse as ul; print(ul.unquote_plus(sys.stdin.read()))'";
      urlencode = "python3 -c 'import sys, urllib.parse as ul; print(ul.quote_plus(sys.stdin.read()))'";
    };
  };

  # This value determines the Home Manager release that your
  # configuration is compatible with. This helps avoid breakage
  # when a new Home Manager release introduces backwards
  # incompatible changes.
  #
  # You can update Home Manager without changing this value. See
  # the Home Manager release notes for a list of state version
  # changes in each release.
  home.stateVersion = "23.05";

  # Let Home Manager install and manage itself.
  programs.home-manager.enable = true;
}
```

添加好 `/etc/nixos/home.nix` 后，还需要在 `/etc/nixos/flake.nix` 中导入该配置，它才能生效，可以使用如下命令，在当前文件夹中生成一个示例配置以供参考：

```shell
nix flake new example -t github:nix-community/home-manager#nixos
```

调整好参数后的 `/etc/nixos/flake.nix` 内容示例如下：

```nix
{
  description = "NixOS configuration";

  inputs = {
    nixpkgs.url = "github:nixos/nixpkgs/nixos-unstable";
    home-manager.url = "github:nix-community/home-manager";
    home-manager.inputs.nixpkgs.follows = "nixpkgs";
  };

  outputs = inputs@{ nixpkgs, home-manager, ... }: {
    nixosConfigurations = {
      # 这里的 nixos-test 替换成你的主机名称
      nixos-test = nixpkgs.lib.nixosSystem {
        system = "x86_64-linux";
        modules = [
          ./configuration.nix

          # 将 home-manager 配置为 nixos 的一个 module
          # 这样在 nixos-rebuild switch 时，home-manager 配置也会被自动部署
          home-manager.nixosModules.home-manager
          {
            home-manager.useGlobalPkgs = true;
            home-manager.useUserPackages = true;

            # 这里的 ryan 也得替换成你的用户名
            # 这里的 import 函数在前面 Nix 语法中介绍过了，不再赘述
            home-manager.users.ryan = import ./home.nix;

            # 使用 home-manager.extraSpecialArgs 自定义传递给 ./home.nix 的参数
            # 取消注释下面这一行，就可以在 home.nix 中使用 flake 的所有 inputs 参数了
            # home-manager.extraSpecialArgs = inputs;
          }
        ];
      };
    };
  };
}
```

然后执行 `sudo nixos-rebuild switch` 应用配置，即可完成 home-manager 的安装。

安装完成后，所有用户级别的程序、配置，都可以通过 `/etc/nixos/home.nix` 管理，并且执行 `sudo nixos-rebuild switch` 时也会自动应用 home-manager 的配置。

`home.nix` 中 Home Manager 的配置项有这几种查找方式：

- [Home Manager - Appendix A. Configuration Options](https://nix-community.github.io/home-manager/options.html): 一份包含了所有配置项的列表，建议在其中关键字搜索。
  - [Home Manager Option Search](https://mipmip.github.io/home-manager-option-search/): 一个更方便的 option 搜索工具。
- [home-manager](https://github.com/nix-community/home-manager): 有些配置项在官方文档中没有列出，或者文档描述不够清晰，可以直接在这份 home-manager 的源码中搜索阅读对应的源码。

## Home Manager vs NixOS

有许多的软件包或者软件配置, 既可以使用 NixOS Module 配置(`configuration.nix`)，也可以使用 Home Manager 配置(`home.nix`), 这带来一个选择难题: **将软件包或者配置文件写在 NixOS Module 里还是 Homa Manager 配置里面有何区别? 该如何决策?**

首先看看区别, NixOS Module 中安装的软件包跟配置文件都是整个系统全局的, 全局的配置通常会被存放在 `/etc` 中, 而全局的软件也通常被链接到全局. 不论切换到哪个用户下,
都能正常使用该软件或配置.

而 Home Manager 安装的所有东西, 都仅在对应的用户下可用, 切换到其他用户后这些配置跟软件就都失效了.

根据这种特性, 一般的推荐用法是:

- NixOS Module: 安装系统的核心组件, 以及几乎所有用户(包含 root 用户)都需要用到的其他软件包.
  - 比如说如果你希望某个软件包能在你切换到 root 用户时仍能正常使用, 或者使某个配置能在 root 用户下也能生效, 那就得用 NixOS Module
    来安装它.
- Home Manager: 其他所有的配置与软件, 都建议用 Home Manager 来安装.
