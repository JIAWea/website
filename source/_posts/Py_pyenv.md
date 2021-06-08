---
title: Python版本管理
date: 2021-06-06 15:58:52
categories: 后端
tags: [python,版本管理,虚拟环境]
---

笔者在使用 `Python` 使通常会遇到当前正在使用的版本与想要用的版本不一致，但是又不能改动它，不然原先使用的应用可能就会无缘无故因为版本问题出现一系列的问题。所以这时候就需要用到了版本控制，笔者使用的是 `pyenv` 来管理不同的版本，`virtualenv` 来创建不同的虚拟环境。

<!--more-->

## 安装

###  脚本安装

可能网络问题安装不成功
```shell
$ sudo curl -L https://raw.githubusercontent.com/yyuu/pyenv-installer/master/bin/pyenv-installer | bash
```


### 手动安装
```shell
git clone https://github.com/yyuu/pyenv.git ~/.pyenv

# 使用 bash 命令行则加入到 ~/.bashrc 文件末尾，并执行 
export PYENV_ROOT="$HOME/.pyenv"
export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init -)"

source ~/.bashrc

# 安装指定版本，使用国内源
v=3.7.9;wget https://npm.taobao.org/mirrors/python/$v/Python-$v.tar.xz -P ~/.pyenv/cache/;pyenv install $v

# 如果缺少依赖
sudo apt-get install -y gcc make build-essential libssl-dev zlib1g-dev libbz2-dev \
libreadline-dev libsqlite3-dev wget curl llvm libncurses5-dev libncursesw5-dev \
xz-utils tk-dev libffi-dev liblzma-dev libldap2-dev libsasl2-dev

# 安装 virtualenv 插件
git clone https://github.com/yyuu/pyenv-virtualenv.git ~/.pyenv/plugins/pyenv-virtualenv

eval "$(pyenv virtualenv-init -)"

source ~/.bashrc
```


常用命令
```shell
# 查看已安装的版本
root@ray:~# pyenv versions
* system (set by /root/.pyenv/version)
  3.7.9

# 切换版本
root@ray:~# pyenv global 3.7.9

# 创建虚拟环境
root@ray:~# pyenv virtualenv test
root@ray:~# pyenv activate test
root@ray:~# pyenv deactivate
```
