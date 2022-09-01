## Mac 安装 fabric 记录

### 1. 安装 brew 

brew 是 Mac 下的一个包管理工具，作用类似于 `centos` 下的 `yum`。

安装brew也很简单，一条命令即可:

```
Copy/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```



### 2. 安装 pip

pip是常用的python包管理工具

安装命令：

```undefined
sudo easy_install pip
```



### 3. 安装 fabric

python自动化运维工具 fabric

安装命令

```
pip install fabric
```



### 遇坑

- Consider adding this directory to PATH or, if you prefer to suppress this warning, use --no-warn-script-location.

将 提示的 /Users/zonst/Library/Python/2.7/bin 添加到 PATH 中

vim ~/.bash_profile

修改

source ~/.bash_profile