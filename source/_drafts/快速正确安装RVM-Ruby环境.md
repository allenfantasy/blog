title: '快速正确安装RVM & Ruby环境'
tags:
---

### MAC OS X

#### Homebrew

```shell
ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```
#### RVM

```shell
curl -L https://get.rvm.io | bash -s stable
source ~/.rvm/scripts/rvm
```

检查是否安装正确：

```
rvm -v
```

#### Ruby

```
rvm install 2.1.3 # 安装当前稳定的版本2.1.3
rvm use 2.1.3 --default
```

测试是否安装正确：

```
ruby -v
gem -v
```

### Linux [TODO]
