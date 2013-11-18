# centosのvagrant boxを作成する手順

rednat系はほぼ共通だと思います。

ここではCentOS6.4(x86_64)を例に取ります。

## VirtualBoxのインストール

[ダウンロードページ](https://www.virtualbox.org/wiki/Downloads)から最新版をダウンロード＆インストールします。

ただし、少なくともVagrant1.3.5とVirtualBox4.3の組み合わせは不具合があるので、
https://www.virtualbox.org/wiki/Download_Old_Builds から4.2を入れましょう。

## Vagrantのインストール

http://downloads.vagrantup.com/ から最新版をダウンロード＆インストールします。

## isoのダウンロード

http://ftp.jaist.ac.jp/pub/Linux/CentOS/5.10/isos/ 或いは http://mirror.symnds.com/distributions/CentOS-vault/ から入れたいバージョンのisoをダウンロードしておきます。

今回は`CentOS-6.4-x86_64-netinstall.iso`を使います。

6系ならminimalでもいいでしょう。


## ベースとなるVMの作成

### 新規

VitualBoxを起動します。

新規ボタンをクリックし、

```
名前：なんでもいいですがcentos64とします
タイプ：Linux
バージョン：Red Hat (64 bit)
```

にして次へ。

あとはデフォルトでいいと思いますが、メモリやディスクの容量など適宜変えて下さい。

### デバイスの設定

USBはオーディオが必要ない場合は無効にしてかまいません。

ネットワークはデフォルト(NATがひとつ有効)でいいです。

ストレージ→コントローラー：IDE→空になっているCDのところを選択して、属性のところのCDのアイコンをクリック。
「仮想CD/DVDディスクファイルの選択」を選択し、先ほどダウンロードしたisoファイルを選択します。

OKをクリックして、VMを起動します。

### Linuxのインストール

"Install or upgrade an existing system"を選択。

以下特記事項のみ。

インストール元は`http://ftp.jaist.ac.jp/pub/Linux/CentOS/6.4/os/x86_64/`などを使いましょう。

rootのパスワードは`vagrant`にします。

パッケージ選択ができる場合は最低限にします。

### 各種設定

リブートしてrootでログインします。

#### ファイヤーウォールの停止

ホスト側からのアクセス制限を解除します。

```
# /etc/init.d/iptables stop
# chkconfig iptables off
```

#### selinuxの無効化

いろいろ縛りがキツくなるので無効化しておきます。

`/etc/sysconfig/selinux`を編集し、

```diff
-SELINUX=enforcing
+SELINUX=disabled
```

に変更する。

#### VirtualBox Guest Additionsのインストール

ゲストとホストの連携を助ける(共有フォルダとか)カーネルモジュールをインストールします。

VirtualBoxのデバイスメニューから、`Guest Additionsのインストール`を選択

コンソールから、

```
# yum update -y kernel
# reboot
# mount /dev/cdrom /mnt
# yum install -y make kernel-headers kernel-devel gcc perl
# /mnt/VBoxLinuxAdditions.run --nox11
```

X11関係のエラーは無視して大丈夫です。

#### vagrantユーザーのセットアップ

ユーザー名＆パスワードがvagrantのユーザーを作ります。
特定の鍵でパスワードなしログインができて、sudoもパスワードなしでできるようにします。

```
# useradd vagrant
# passwd vagrant
# groupadd admin
# usermod -a -G admin vagrant
# visudo
```

次のように編集します。

```
env_keepにSSH_AUTH_SOCKを追加
Defaults requirettyをコメントアウト
%admin ALL=NOPASSWD: ALLを追加
```
sshの鍵の設定をします。

```
# yum install -y openssh-clients
# cd /home/vagrant
# mkdir .ssh
# cd .ssh
# curl https://raw.github.com/mitchellh/vagrant/master/keys/vagrant.pub > authorized_keys
# chown -R vagrant:vagrant .
# chmod 700 .
# chmod 600 authorized_keys
```

#### chefのインストール

chefはruby 1.9以降が必要ですがcentos6はruby1.8なので、ここではソースか
ら入れます。

[公式ダウンロードページ](https://www.ruby-lang.org/ja/downloads/)から
ruby2.0のソースをダウンロードしてきて展開し、
```
yum install -y openssl-devel
configure --prefix=/usr --disable-install-doc && make && make install
```
でインストールします。

終わったらソースは消しましょう。

次に`gem install chef`してchefを入れます。

#### その他必要なもののインストール

他に必要なものがあれば入れて下さい。

基本的に各プロジェクトで使用するものは、provisionの方で個別に入れるようにしましょう。

## パッケージング

ゲストOSをhaltし、VirtualBoxのGUIを終了します。

ホスト側のコマンドラインから次のコマンドを実行します。

```
vagrant package --base centos64
```

するとpackage.boxができます。
