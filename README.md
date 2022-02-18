

### fork三个仓库，并对三个仓库做修改

注意：需要选取一个名称作为编译的iso名称，不能选用**pfSense**，这里选择**libreSense**


fork下面三个仓库: 
- [pfSense GUI](https://github.com/pfsense/pfsense)
- [FreeBSD Ports](https://github.com/pfsense/freebsd-ports) (Ports collection is the FreeBSD package management system, it works like `apt` or `rpm` on Linux)
- [FreeBSD Kernel Source](https://github.com/pfsense/freebsd-src)

fork 完成后对三个仓库做如下修改

### FreeBSD Source
- Checkout to the the branch you would like to build (`devel-12` for dev version, `RELENG_2_5_0` for stable version).
- In the folder `/release/conf/`, rename `pfSense_src-env.conf`, `pfSense_src.conf` and `pfSense_make.conf` to `libreSense_src-env.conf`, `libreSense_src.conf` and `libreSense_make.conf`
- Rename the file `/sys/amd64/conf/pfSense` to `/sys/amd64/conf/libreSense`

### FreeBSD Ports
- Checkout to the the branch you would like to build (`devel` for dev version, `RELENG_2_5_0` for stable version).
-  Edit the file `/security/pfSense/Makefile` : remove `wireguard-tools>=0:security/wireguard-tools` from the `RUN_DEPENDS` variable. (While [open source](https://github.com/pfsense/wireguard-tools), the wireguard-tools port for pfSense is currently not integrated into FreeBSD)

### pfSense GUI
- Go to the folder `/tools/templates/pkg_repos/` in the branch you would like to build (`master`for dev version, `RELENG_2_5_0` for stable version) and change `pfSense` to `libreSense` in the file names (e.g., `pfSense-repo.abi => libreSense-repo.abi`)
- Edit the file `/src/etc/inc/globals.inc` : replace the content of `product_name` by `libreSense`, and the content of `pkg_prefix` by `libreSense-pkg-`.
- In the folder `/src/usr/local/share/`, rename the folder `pfSense` to `libreSense`.
- In the folder `/src/etc/`, rename the files `pfSense-ddb.conf` and `pfSense-devd.conf` to `libreSense-ddb.conf` and `libreSense-devd.conf`.
- Edit the file `/tools/builder_defaults.sh` : remove `if_wg` from the `MODULES_OVERRIDE` variable. 

## 配置编译环境

编译使用的freeBSD至少需要10G内存，8核以上cpu。cpu核心数越多编译越快。另外需要**能够访问国外资源**。

**本文使用freeBSD12.2  16G内存   48核CPU  60G磁盘（如果是固态硬盘更好，编译过程中有大量的磁盘IO) 编译分支为RELENGV2_5_0**

在安装freeBSD时，需要选择ZFS文件格式，其它步骤正常安装即可。

![Disk partition](
https://github.com/Augustin-FL/building-pfsense-iso-from-source/blob/master/images/ZFS.png?raw=true)

安装完成后依次输入以下命令配置环境。（若下载速度慢，则为网络不能翻墙）

```
# Allow SSH using root, if you want it.
echo PermitRootLogin yes >> /etc/ssh/sshd_config
service sshd restart

# Required for configuring the server
pkg install -y pkg vim nano emacs

# Required for installing and building ports
pkg install -y git nginx poudriere-devel rsync sudo

# Required for building kernel and iso
pkg install -y vmdktool curl qemu-user-static gtar xmlstarlet pkgconf openssl

# Required for building iso
portsnap fetch extract

# not required but advised for building/monitoring/debugging
pkg install -y htop screen wget mmv

# （只有是虚拟机才需要安装）
pkg install -y open-vm-tools

# （内存如果够大，不需要这一步）
dd if=/dev/zero of=/root/swap.bin bs=1M count=8192
chmod 0600 /root/swap.bin
mdconfig -a -t vnode -f /root/swap.bin -u 0 
echo 'swapfile="/root/swap.bin"' >> /etc/rc.conf
swapon /dev/md0
```



接下来配置Nginx:
```
# pfSense_gui_branch represents the branch of pfSense GUI that will be compiled, with "RELENG" replaced by "v" : master for a development ISO, v2_5_0 for a stable ISO
# pfSense_port_branch represents the branch of FreeBSD ports that will be compiled, using the same replacement ("RELENG"=>"v") : devel for a development ISO, v2_5_0 for a stable ISO
# product_name represents the name of your product.

set pfSense_gui_branch=v2_5_0 # Replace with the version you want to build
set pfSense_port_branch=v2_5_0 # Replace with the version you want to build
set product_name=libreSense # Replace with your product name


cd /usr/local/www/nginx/
rm -rf *
mkdir -p packages
# Ports web server for core PKGs (pfSense-base, pfSense-rc, etc...)
ln -s /root/pfsense/tmp/${product_name}_${pfSense_gui_branch}_amd64-core/.latest packages/${product_name}_${pfSense_gui_branch}_amd64-core
# Ports web server for other PKGs
ln -s /usr/local/poudriere/data/packages/${product_name}_${pfSense_gui_branch}_amd64-${product_name}_${pfSense_port_branch} packages/${product_name}_${pfSense_gui_branch}_amd64-${product_name}_${pfSense_port_branch} 
# Web server for monitoring ports build
ln -s /usr/local/poudriere/data/logs/bulk/${product_name}_${pfSense_gui_branch}_amd64-${product_name}_${pfSense_port_branch}/latest poudriere

# Allow directory indexing, and configure nginx to start automatically on boot
sed -i '' 's+/usr/local/www/nginx;+/usr/local/www/nginx; autoindex on;+g' /usr/local/etc/nginx/nginx.conf
echo nginx_enable=\"YES\" >> /etc/rc.conf
service nginx restart
```

## 生成签名密钥

```
mkdir -p /root/sign/
cd /root/sign/
openssl genrsa -out repo.key 2048
chmod 0400 repo.key
openssl rsa -in repo.key -out repo.pub -pubout
printf "function: sha256\nfingerprint: `sha256 -q repo.pub`\n" > fingerprint
```

在 `/root/sign/` 创建`sign.sh`包含以下内容
```
#!/bin/sh
read -t 2 sum
[ -z "$sum" ] && exit 1
echo SIGNATURE
echo -n $sum | openssl dgst -sign /root/sign/repo.key -sha256 -binary
echo
echo CERT
cat /root/sign/repo.pub
echo END
```
最后，给sign.sh执行权限
```
chmod +x /root/sign/sign.sh
```


接下来配置如何编译pfsense
```
cd /root
git clone https://github.com/{your username}/pfsense.git
cd pfsense
git checkout RELENG_2_5_0 # Replace by the branch of pfSense GUI to build.

# Ports repositories signing key
rm src/usr/local/share/${product_name}/keys/pkg/trusted/*
cp /root/sign/fingerprint src/usr/local/share/${product_name}/keys/pkg/trusted/fingerprint
```


在`/root/pfsense`目录（也就是刚才克隆的仓库）下创建build.conf文件，包含以下内容。注意：不要在Windows上创建`build.conf`然后上传到服务器，尽量在服务器上直接使用`touch`和`vim`来操作，否则可能会编译报错。
```
export PRODUCT_NAME="libreSense" # Replace with your product name
export FREEBSD_REPO_BASE=https://github.com/{your username}/FreeBSD-src.git # Location of your FreeBSD sources repository
export POUDRIERE_PORTS_GIT_URL=https://github.com/{your username}/FreeBSD-ports.git # Location your FreeBSD ports repository

export FREEBSD_BRANCH=RELENG_2_5_0 # Branch of FreeBSD sources to build

# The branch of FreeBSD ports to build is set automatically based on pfSense GUI branch.
# If you would like to build a specific branch of FreeBSD ports, the variable to set is POUDRIERE_PORTS_GIT_BRANCH
# Also, if you are building a FreeBSD port branch that does not respect Netgate conventions (devel or RELENG_*),
# You will also have to update .conf files in "tools/templates/pkg_repos/" because repositories names are partially 
# hardcoded there.



# Netgate support creation of staging builds (pre-dev, nonpublic version)
unset USE_PKG_REPO_STAGING # This disable staging build
# The kind of ISO that will be built (stable or development) is defined in src/etc/version in pfSense GUI repo

export DEFAULT_ARCH_LIST="amd64.amd64" # We only want to build an x64 ISO, we don't care of ARM versions

# Signing key
export PKG_REPO_SIGNING_COMMAND="/root/sign/sign.sh ${PKG_REPO_SIGN_KEY}"

# This command retrieves the IP address of the first network interface
export myIPAddress=$(ifconfig -a | grep inet | head -1 | cut -d ' ' -f 2)

export PKG_REPO_SERVER_DEVEL="pkg+http://${myIPAddress}/packages"
export PKG_REPO_SERVER_RELEASE="pkg+http://${myIPAddress}/packages"

export PKG_REPO_SERVER_STAGING="pkg+http://${myIPAddress}/packages" # We need to also specify this variable, because even
# if we don't build staging release some ports configuration is made for staging.
```

# 开始编译

使用 ```screen -S build``` 打开一个窗口 使用 ctrl+A then D可以离开窗口，使用 ```screen -r build``` 可以重新进入窗口。这样免得ssh断开连接就停止编译了。

### Setup Jails
执行 ```./build.sh --setup-poudriere```. 


### Build ports
执行 ```./build.sh --update-pkg-repo``` 
在浏览器 ( http://ipOfYourServer/poudriere )可以查看编译进度（如果查看不了，前面配置nginx有问题）

有些port编译会失败，原因可能是该port的版本低不再受支持。

本文在编译时，把失败的port都给删除了，这样就不会再出现编译失败了。。。 [poudriere_bulk](https://github.com/pfsense/pfsense/blob/master/tools/conf/pfPorts/poudriere_bulk) （在这里删除，这里面没有所有的Port，因为port有依赖关系，这里只有最上层的port，当需要删除的port不再文件中时，需要查清楚依赖关系，看是哪个上层port依赖该port）

在该阶段最后需要签名，可能出现失败情况。signing repo failed...

解决方法：

1. 在开始编译之前查看pkg版本使用`pkg --v`如果为1.17则需要降级到1.16。

   ```shell
   pkg install portdowngrade
   cd /root
   portdowngrade ports-mgmt/pkg rc565958
   然后根据输出结果完成pkg安装
   ```

2. 在build.sh中配置跳过签名阶段

   在`build.sh`第约150左右添加一行代码

   ```
   --update-pkg-repo)
   export DO_NOT_SIGN_PKG_REPO=YES //添加这一行
   BUILDACTION="update_pkg_repo"
   ;;
   ```

   

### Build kernel and create ISO

 最后执行 `./build.sh --skip-final-rsync iso`

At the end of the build, a compressed iso file (`.iso.gz`) will be present in `~pfsense/tmp/${product_name}/installer/`. You can extract it using `gzip -kd *.gz` if you need the plain `.iso`.

在该阶段可能出现问题即解决办法：

1. 在中间有一个make阶段报错。若编译环境选择48核。执行make -j96会报错。

   解决办法：减少cpu核心数到24核即可解决。

2. installing ports阶段报错。

   报错原因为在安装ports时url出错。

   解决办法：

   ```
   # cd /usr/local/www/nginx-dist/packages
   # ln -s /usr/local/poudriere/data/packages/libreSense_v2_5_0_amd64-libreSense_v2_5_0 libreSense_v2_5_1_amd64-libreSense_v2_5_1
   (注意：第二条命令较长，中间有空格)
   ```

   

   