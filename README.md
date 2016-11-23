# development-enviroment-building
vagrant + vritualbox + cenos + apache + tomcat + gitbucket + jenkins + redmine + artifactory  の vagrant box を作る。

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

1. 下記サイトから取得、配置する。

 ~~~bash
 $ cd /var/lib/tomcat/webapps
 $ wget http://mirrors.jenkins.io/war-stable/latest/jenkins.war
 $ chmod 777 jenkins.war
 ~~~

## ファイアウォールの設定
1. 80番ポートを開ける。

  ~~~bash
  $ firewall-cmd --add-port=80/tcp --permanent
  ~~~

## Apacheのプロキシ設定
1. Apacheのプロキシファイルを作成する。

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

1. apacheを再起動する。

   ~~~bash
   $ systemctl restart httpd
   ~~~

1. VirtualBoxのNATポートフォワーディング設定を行う。

  > ファイル → 環境設定 → ネットワーク → NATネットワーク「新しいNATネットワーク追加」 → NATNetworkをダブルクリック → ポートフォワーディング

| 設定箇所       | 設定値        |
|:--------------|:--------------|
| ホストポート | 8080         |
| ゲストIP     | 192.168.33.10 |
| ゲストポート | 80            |

1. ホストPCブラウザからアクセス確認する。

  > http://localhost:8080/gitbucket
  > http://localhost:8080/jenkins

## redmineのインストール

  1. Ansibleとgitのインストールする。

     ~~~bash
     $ yum install -y epel-release
     $ yum install -y ansible git
     ~~~

  1. 任意の場所にredmine-centos-ansibleを取得する。

     ~~~bash
     $ cd
     $ mkdir redmine
     $ cd redmine
     $ git clone https://github.com/farend/redmine-centos-ansible.git
     ~~~

  1. パスワードを変更し、インストールする。

     ~~~bash
     $ cd redmine-centos-ansible
     $ vim group_vars/redmine-servers
     $ ansible-playbook -i hosts site.yml
     ~~~

  1. ホストPCブラウザからアクセス確認する。

    > http://localhost:8080/redmine


  ### redmineインストール時に発生したエラーの対処方法

  * rubyのソースがダウンロードできない！

    > /tmp/redmine-setupにwgetで取得。

    ~~~bash
    $ cd /tmp/redmine-setup
    $ wget https://cache.ruby-lang.org/pub/ruby/2.2/ruby-2.2.3.tar.bz2
    ~~~
    > ソースコードをダウンロードしないようにコメントアウト

    ~~~bash
    $ cd redmine-centos-ansible/
    $ vim roles/ruby/tasks/main.yml
    ~~~
    以下をコメントアウト

    ~~~
    #- name: Rubyのソースコードのダウンロード
    #  get_url:
    #    url={{ ruby_url_dir }}/{{ ruby_archive_name }}
    #    dest={{ work_dir }}
    #  when: ruby_test_vars.rc == 1 and ruby_src_test_vars.rc == 1  
    ~~~

    > 再実行

  * SVNチェックアウトができない！（Unknown Host）

    > subversionにプロキシが設定されていないため

    ~~~bash
    $ vim .subversion/servers
    ~~~

    [global]の以下２行のコメントアウトを外して、設定する。

    ~~~bash
    [global]
    # http-proxy-exceptions = *.exception.com, www.internal-site.org
    http-proxy-host = proxy.example.com
    http-proxy-port = 8080
    # http-proxy-username = defaultusername
    ~~~

    > 再実行

## Subversionのインストールと設定

  1. subversionがインストール済か確認する。（インストール済のバージョン等が表示される。）

    ~~~bash
    $  yum list installed | grep subversion
    ~~~

  1. mod_dav_svn のインストールする。

    ~~~bash
    $ yum install -y mod_dav_svn
    ~~~

  1. apacheのconfを作成する。

    ~~~bash
    $ vim /etc/httpd/conf.d/subversion.conf
    ~~~

    subversion.confの内容

    ~~~
    LoadModule dav_svn_module     modules/mod_dav_svn.so
    LoadModule authz_svn_module   modules/mod_authz_svn.so
    LoadModule dontdothat_module  modules/mod_dontdothat.so

    #Alias /svn /var/www/svn
    <Location /svn>
      DAV svn
      SVNParentPath /var/www/svn/
    </Location>
    ~~~

  1. レポジトリの作成する。（作成する名前はなんでも良いがRedmineの識別子と同じものにする：compasとしている）

    ~~~bash
    $ mkdir /var/www/svn
    $ cd /var/www/svn
    $ svnadmin create repo
    $ chown apache:apache repo
    ~~~

  1. apacheを再起動する。

    ~~~bash
    $ systemctl restart httpd
    ~~~

  1. ホストPCブラウザからアクセス確認する。

    > http://localhost:8080/svn/compas


## RedmineとSubversionを連携する。（RedmineアカウントでSVNの認証を行う）

  1. mod_perlとPerlモジュールをインストールする。

    ~~~bash
    $ yum install -y epel-release
    $ yum install -y mod_perl
    $ yum install -y perl-DBI perl-DBD-Pg perl-Digest-SHA
    ~~~

  1. mod_perlから見えるところにRedmine.pmを配置する。

    ~~~bash
    $ mkdir -p /usr/lib64/perl5/vendor_perl/Apache/Authn
    $ ln -s /var/lib/redmine/extra/svn/Redmine.pm /usr/lib64/perl5/vendor_perl/Apache/Authn/Redmine.pm
    ~~~

  1. subversion.confを変更する。

    ~~~bash
    $ vim /etc/httpd/conf.d/subversion.conf
    ~~~

    subversion.confの内容

    ~~~
    LoadModule dav_svn_module     modules/mod_dav_svn.so
    LoadModule authz_svn_module   modules/mod_authz_svn.so
    LoadModule dontdothat_module  modules/mod_dontdothat.so

    PerlLoadModule Apache::Authn::Redmine

    #Alias /svn /var/www/svn
    <Location /svn>
      DAV svn
      SVNParentPath /var/www/svn/
      AuthType Basic
      AuthName "SVN Repository"
      AuthUserFile /dev/null
      Require valid-user

      PerlAuthenHandler Apache::Authn::Redmine::authen_handler

      RedmineDSN "DBI:Pg:dbname=redmine;host=localhost"
      RedmineDbUser "redmine"
      RedmineDbPass "redmine" #redmineインストール時に変更したパスワードを設定
    </Location>
    ~~~

  1. apacheを再起動する。

    ~~~bash
    $ systemctl restart httpd
    ~~~

  1. Redmineにアカウント管理用のプロジェクトを作成する。

    認証時に使用されるアカウントは、Subversion のパッケージ名と一致する識別子を持つプロジェクトのメンバーであることが必要。
    Redmineで識別子を compas とするプロジェクトを作成し、Subversionにアクセスするユーザーをメンバーに登録します。
    http://localhost/svn/repo にアクセスし、Redmineのアカウントで認証できることを確認します。


## Artifactoryのインストール

  1. JAVA_HOMEを設定する。

    ~~~bash
    $ vim /etc/bashrc
    ~~~

    bashrcの内容（JAVAは入れたバージョンによって異なるので確認すること）

    ~~~
    export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.111-1.b15.el7_2.x86_64
    export PATH=$PATH:$JAVA_HOME/bin
    export CLASSSPATH=.:$JAVA_HOME/lib/tool.jar:$JAVA_HOME/lib/dt.jar
    ~~~

    ~~~bash
    $ source /etc/bashrc
    ~~~

  1. Artifactoryのページからzipをダウンロードする。

    > https://www.jfrog.com/open-source/

  1. WinSCPでvagrantの/tmpにアップロードし解凍する。

    ~~~bash
    $ mkdir /opt/artifactory
    $ mv /tmp/jfrog-artifactory-oss-4.14.1.zip /opt/artifactory
    $ cd /opt/artifactory
    $ unzip jfrog-artifactory-oss-4.14.1.zip
    ~~~

  1. インストールする。

    ~~~bash
    $ cd /opt/artifactory/artifactory-oss-4.14.1/bin
    $ ./installService.sh
    ~~~

  1. apacheのプロキシ設定をする。


     ~~~bash
     $ vi /etc/httpd/conf.d/artifactory.conf
     ~~~

     artifactory.conf

     ~~~
     <Location /artifactory>
         ProxyPass ajp://localhost:8019/artifactory
     </Location>
     ~~~

  1. ホストPCブラウザからアクセス確認する。

    > http://localhost:8080/artifactory
