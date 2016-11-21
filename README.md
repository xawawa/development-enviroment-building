# development-enviroment-building
vagrant + vritualbox + cenos + apache + tomcat + gitbucket + jenkins + redmine + artifactory  の vagrant box を作る。

1. gitをインストールする。

 > https://git-scm.com/downloads/

1. git及びgit-bashのパスを通す。

 > ${Gitのインストールディレクトリ}\\cmd

 > ${Gitのインストールディレクトリ}\\usr\\bin

1. VirtualBoxをインストールする。

 > https://www.virtualbox.org/

1. Vagrantをインストールする。

 > https://www.vagrantup.com/downloads.html

1. Vagrantファイルを格納するディレクトリを作成する。

   ~~~bash
   $ mkdir /vagrant
   ~~~

1. Vagrantのboxを以下のサイトから選択して追加する。

 > http://www.vagrantbox.es/

   ~~~bash
   $ vagrant box add centos7 https://github.com/tommy-muehle/puppet-vagrant-boxes/releases/download/1.1.0/centos-7.0-x86_64.box
   ~~~

1. Vagrant を初期化する。(Vagrantfileが作成される。)

  ~~~bash
  $ cd /vagrant
  $ vagrant init centos7
  ~~~

1. Vagrantfileを編集する。

   ~~~
   # 以下の行のコメントアウトを外す。
   config.vm.network "forwarded_port", guest: 80, host: 8080

   # 以下の行のコメントアウトを外す。(hostから繋ぐときのIPアドレスとなる)
   config.vm.network "private_network", ip: "192.168.33.10"

   # 以下の行のコメントアウトを外し、VirtualBoxが使用するメモリを2Gにしておく（VirtualBoxで立ち上げる設定を有効にする）
   config.vm.provider "virtualbox" do |vb|
   vb.gui = true
   vb.memory = "2048"
   end
   ~~~

1. CentOS7を起動する。

  ~~~bash
  $ vagrant up
  ~~~

  起動しない場合はBIOSの「仮想化支援機能」が「無効」となっている可能性があるので、起動時にF10等を押下して、BIOS画面を表示させて「有効」に変更する。

1. 起動したCentOS7にSSHログインする。（rootになっておく、pass:vagrant）

   ~~~bash
   $ vagrant ssh
   $ su -
   ~~~

1. リポジトリを追加する。

   ~~~bash
   $ yum install -y epel-release
   $ rpm -ivh http://rpms.famillecollet.com/enterprise/remi-release-7.rpm
   ~~~

1. パッケージをインストールする。

   ~~~bash
   $ yum --enablerepo=epel,remi install -y sudo vim-enhanced syslog httpd httpd-devel php php-devel php-pear php-mysql php-gd php-mbstring php-pecl-imagick mariadb-server wget git java-1.8.0-openjdk-devel.x86_64 tomcat
   ~~~

1. 各種サービスを起動する。

  ~~~bash
  $ systemctl start httpd
  $ systemctl enable httpd
  $ systemctl start tomcat
  $ systemctl enable tomcat
  $ systemctl start mariadb
  $ systemctl enable mariadb
  ~~~

## Gitbucketをインストール

1. 下記サイトから取得、配置する。

  ~~~bash
  $ cd /var/lib/tomcat/webapps
  $ wget https://github.com/gitbucket/gitbucket/releases/download/4.6/gitbucket.war
  $ chmod 777 gitbucket.war
  ~~~

## Jenkinsをインストール

~~~bash
$ cd /var/lib/tomcat/webapps
$ wget http://mirrors.jenkins.io/war-stable/latest/jenkins.war
$ chmod 777 jenkins.war
~~~

## ファイアウォールの設定

  ~~~bash
  $ firewall-cmd --add-port=80/tcp --permanent
  ~~~

## Apacheのプロキシ設定

  ~~~bash
  $ vim /etc/httpd/conf.d/gitbucket.conf
  ~~~

gitbucket.conの内容

  ~~~
  <Location /gitbucket>
    ProxyPass ajp://localhost:8009/gitbucket
  </Location>
  ~~~

  ~~~bash
  $ vim /etc/httpd/conf.d/jenkins.conf
  ~~~

jenkins.conの内容

  ~~~
  <Location /jenkins>
    ProxyPass ajp://localhost:8009/jenkins
  </Location>
  ~~~
