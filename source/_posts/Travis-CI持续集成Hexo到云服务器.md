---
title: Travis-CI 持续集成 hexo 到云服务器
date: 2019-09-01 18:35:40
categories: 
- devOps
tags:
- travis
- CI
- hexo
- gem
---

> 烦：每次本地提交代码到 GitHub 上后还要自己手动同步文件到云服务器，完成所谓的部署工作有点麻烦和脱节。
> 懒：程序员等所有自称工程师进步的先决动力，想实现一个我每次提交代码后，后面的构建，打包和部署都自动进行的流程。
> 因为本身代码托管在 Github 上，所以就开始折腾 Travis-CI了。


## 什么是CI
持续集成服务（Continuous Integration，简称 CI）。
一般指的是提供一个运行环境，自动化执行单元测试，规范检查，基于环境的构建，服务部署等流水线作业。
持续集成的目的，就是让产品可以快速迭代，同时还能保持高质量。
另外，这个是工程化实践，如果你觉得自己手动操作比整这些繁琐的流程方便多了，我觉得也是没问题的。

## Travis CI 
[Travis CI](https://travis-ci.com/) 对 Github 上的开源项目提供免费服务，这要求必须有 Github 账号。界面也很赞~，让我想起了前东家的 PLUS 发布系统。 
![travis](/images/travis.png)
可以使用 github 账号直接登录， public 的仓库也直接同步到 Travis 上了， 选择想开启 Travis CI 的仓库，打开开关即可。详细配置不细说了，图形化界面，进去就知道了。

### .travis.yml
Travis-CI配置文件，存放在项目根目录下。
支持多种语言，在配置文件中 `language: node_js`
一个比较完整的生命周期

```bash
before_install
install
before_script
script
aftersuccess or afterfailure
[OPTIONAL] before_deploy
[OPTIONAL] deploy
[OPTIONAL] after_deploy
after_script
```
详细配置参考 [官方文档](https://docs.travis-ci.com)

## 一些准备工作
由于后面的免密登录和部署流程会涉及一些环境和配置类的操作，这个部分介绍下前置条件。

### SCP命令
scp 命令用于linux下的跨主机之间的文件和目录复制
在首次连接服务器时，会弹出公钥确认的提示。这会导致某些自动化任务，由于初次连接服务器而导致自动化任务中断，
可在 StrictHostKeyChecking选项，用 -o 参数指定后，则不检查该项。

```bash
scp [可选参数] file_source file_target
#将public目录下的所有文件复制到$DEPLOY_IP下的/path/to/blog目录中。不检查key，
scp -o StrictHostKeyChecking=no -r public/*  user@$DEPLOY_IP:/path/to/blog/
#可用-i指定私钥。
scp -o  StrictHostKeyChecking=no -i .ssh/id_rsa yourfile user@destinate_ip:/dest_folder
#或将自己的公钥放到目标机的authorized_keys文件里，使自己为目标机的信任机器，实现无密码登录
#这个是在生成ssh key 后，将公钥放到authorized_keys文件中。使用密钥对可以实现不输入密码
cd ~/.ssh
cat id_rsa.pub >> authorized_keys
```

### rsync 命令
最终采用了 rsync 命令，我觉得都行，主要是这个同步成功了。
```bash
rsync -e "ssh -o PubkeyAuthentication=yes  -o stricthostkeychecking=no" -r --delete-after --quiet $TRAVIS_BUILD_DIR/public/* root@$DEPLOY_IP:/opt/hexoBlog
```

### centOS使用gem
因为后面要使用 `gem install travis` ,所以可能会需要 升级 ruby 和 切换 gem 源（亲测 ruby 版本低会安装报错，gem 用官方源真的是动都不动啊，太难了）


#### 切换gem源
```bash
$ gem sources --add https://gems.ruby-china.com/ --remove https://rubygems.org/
$ gem sources -l
https://gems.ruby-china.com
# 确保只有 gems.ruby-china.com
$ gem update --system 
$ gem -v
2.6.3
```
#### 升级Ruby
安装[RAM](https://rvm.io/), 一款ruby版本管理工具，类似 node 的 nvm。
```bash
gpg --keyserver hkp://keys.gnupg.net --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3 7D2BAF1CF37B13E2069D6956105BD0E739499BDB
curl -sSL https://get.rvm.io | bash -s stable
source /etc/profile.d/rvm.sh
rvm -v
```

安装新版 ruby
```bash
rvm install 2.6
```

## 自动部署到远程服务器
对于项目的构建来说都是 执行配置文件中写好的脚本，这个项目可能 就是 `npm install && hexo clean && hexo g`, 那么怎么执行最后一步，把文件同步传输到 云服务器上呢。 我们使用CI就是手动过程太繁琐，重复没有意义，怎么实现 `免密` 部署呢。

登录到 `云服务器(centOS 7.x)`,进行如下操作
### gem install travis 并登录
```bash
gem install travis #  这步失败的话请看上面关于升级ruby和切换gem源的部分
travis login
```
登录 github 账号密码，这个安全直接连接的 github 服务
![login](/images/login.png)

### 生成 ssh key 并输出对应加密的私钥到 travis 
进到云服务器对应的 git 仓库目录里
```bash
ssh-keygen -t rsa -b 4096 -C 'build@travis-ci.org' -f ./deploy_rsa
travis encrypt-file deploy_rsa --add
ssh-copy-id -i deploy_rsa.pub <ssh-user>@<deploy-host>

rm -f deploy_rsa deploy_rsa.pub
git add deploy_rsa.enc .travis.yml
```
项目根目录下的 `deploy_rsa.enc` 文件就是我们加密的私钥文件， `.travis.yml` 是我们的配置文件。
$encrypted_XXXXXX_key 和 $encrypted_XXXXXXXX_iv 是travis 帮忙生成的环境变量，已经同步到 huguobo/hexo-blog 这个项目上了。
![iv](/images/iv.png)

还有一点可能会用上，因为 travis 第一次登录远程服务器会出现 SSH 主机验证，这边会有一个主机信任问题。官方给出的方案是添加 addons 配置，然后修改 .travis.yml 的相关配置
```bash
addons:
  ssh_known_hosts: your-ip

before_deploy:
- openssl aes-256-cbc -K $encrypted_<...>_key -iv $encrypted_<...>_iv -in deploy_rsa.enc -out /tmp/deploy_rsa -d
- eval "$(ssh-agent -s)"
- chmod 600 /tmp/deploy_rsa
- ssh-add /tmp/deploy_rsa
```

最终的部署配置, 我的是静态页面，部署就是同步文件到服务器固定目录，用的是 `rsync`，其中 -e "ssh -o PubkeyAuthentication=yes  -o stricthostkeychecking=no" 参数可以跳过第一次登录的验证。
```bash
deploy:
  provider: script
  skip_cleanup: true
  script: rsync -e "ssh -o PubkeyAuthentication=yes  -o stricthostkeychecking=no" -r --delete-after --quiet $TRAVIS_BUILD_DIR/public/* root@$DEPLOY_IP:/opt/hexoBlog
  on:
    branch: master
```
其他的 yml 配置需要自己根据情况配置了~

## 我最终的 .travis.yml 配置

```bash
language: node_js

node_js:
  - "10"

cache:
  apt: true
  directories:
    - node_modules

addons:
  ssh_known_hosts: $DEPLOY_IP

install:
  - npm install hexo-cli@2.0.0 -g
  - npm install

script:
  - hexo clean 
  - hexo g

before_deploy:
- openssl aes-256-cbc -K $encrypted_25ad2a76f550_key -iv $encrypted_25ad2a76f550_iv -in deploy_rsa.enc -out deploy_rsa -d
- eval "$(ssh-agent -s)"
- chmod 600 deploy_rsa
- ssh-add deploy_rsa

deploy:
  provider: script
  skip_cleanup: true
  script: rsync -e "ssh -o PubkeyAuthentication=yes  -o stricthostkeychecking=no" -r --delete-after --quiet $TRAVIS_BUILD_DIR/public/* root@$DEPLOY_IP:/opt/hexoBlog
  on:
    branch: master

# deploy:
#   provider: script
#   skip_cleanup: true
#   script: scp -o StrictHostKeyChecking=no -r public/*  root@$DEPLOY_IP:/opt/hexoBlog/
#   on:
#      branch: master

branches:
  only:
    - master

notifications:
  email:
    - huguobo2010@126.com
  on_success: change
  on_failure: always
```

## 总结
折腾了大半天，终于看到了 CI-Success 的邮件, 网站也是成功的状态，云主机对应目录的文件也确实更新了。
以后写博客 终于 只用 git push ，其他的等邮件通知了~~
另外 文中的 id_rsa.enc 文件一开始我直接 vim 复制的都是乱码，一开始一直报错 `bad decrypt` ，后来上主机 git clone 仓库直接仓库内生成并添加的，这可能是个坑点。
还有最后用 `rsync` 代替了 `scp`
最后也把 openssl 放在了 before_deploy 阶段，放在 before_install也是没问题的，这些都是因人而异啦。
![success](/images/success.png)

## 参考文章
- https://oncletom.io/2016/travis-ssh-deploy/
- http://www.ruanyifeng.com/blog/2017/12/travis_ci_tutorial.html
- https://gems.ruby-china.com/