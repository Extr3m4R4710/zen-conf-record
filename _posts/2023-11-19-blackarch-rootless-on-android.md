---
layout: post
title: AndroidスマートフォンにBlackArchを構築する。root化不要
data: 2023-11-19 5:31 +0900
tag: [linux, chroot, モバイル, hack]
---
![](https://raw.githubusercontent.com/Extr3m4R4710/zen-conf-record/main/IMG/23-11-19/blackarch-rootless-on-android/blackarch-on-my-phone-with-keybord.jpg)

Androidスマートフォンをペンテスト仕様にカスタムする方法としてNethunterをROM焼きするか、TermuxにNethunter Rootless(nh)を導入する方法が一般的だが、Kali(特にnh rootless)はBlackArchと比較してツール数が少なかったり幅広い開発・侵入用支援モジュールが不足している側面も見受けられる。ユニバーサルなDebianとハッカーフレンドリーなArchLinuxの違い。ほんの出来心でBlackArch環境にトライしてみる。<br>

本稿ではNetHunter Rootlessの代替として機能するBlackArch環境構築の手引きとnh以外の選択肢を示すことにする。<br>

先に自動化スクリプトを公開しておく。これをインストールしたproot ArchLinux環境で実行。<br>
```bash
#!/bin/bash
#
# setup-blackarch.sh
#
#


# 中立かつ安定したDNSに変更
echo "nameserver 1.1.1.1" > /etc/resolv.conf

# パッケージリストの更新と下準備
pacman -Syyu && pacman -S curl base-devel yay

# BlackArch化modを適用
curl -sL http://blackarch.org/strap.sh | bash -

#先に入手できる範囲のツールを取得、取得できない場合はyayで補う
pacman -Syyu && pacman -S nmap sqlmap hydra hashcat recon-ng sn0int sherlock
```
## Termuxの取得
[GitHub](https://github.com/termux/termux-app/releases)でAPKを入手するか、[F-Droid](https://f-droid.org/)から入手する必要がある。<br>

## Prootの準備
pkgコマンドでパッケージのアップグレードとproot-distroをインストールする。確認を求められるのが嫌なら各々`-y`オプションを付ける。 <br>
```bash
pkg upgrade
pkg install proot-distro
```
セットアップが完了したら`proot-distro install archlinux`を実行しルートディレクトリとなる圧縮ファイルをダウンロードする。<br>
prootはchrootのtermux実装でchrootと同じくルートディレクトリ構造をファイルシステムと認識しその中で仮想的なOSとして振る舞う。 <br>
その為xz形式(だっけ？)を展開する。<br>

ArchLinuxのインストールが完了すれば`proot-distro login archlinux`でログイン。コマンドラインとしてコピペでも構わないが、その後先程のシェルスクリプトを実行する。<br>
```bash
# スクリプトとして実行する場合。ファイルに落とし込む場合は要CUIエディタ
chmod +x <任意のファイル名>.sh
```

blackarchレポジトリにアクセスできない場合、`/etc/pacman.d/blackarch-mirrorlist`を適宜修正する。例えばKDDIの日本サーバーのコメントアウトを外すなど。<br>

## ユーザーの作成
ルートでは動かないyayやnmapを使うため、非rootユーザーを作成する。その為のbase-devel。 <br>
### 手順
まずホームディレクトリとログインシェルと同時にユーザーを設定する。 <br>
```bash
 useradd -m -s /bin/bash username # ユーザー名とシェルは好みに合わせて変更すること
```
一旦rootに戻どりパスワードを設定する。 <br>
```bash
passwd username
```

## PKGBUILD上の制約でインストールできないツールをyayで補う

ArchLinuxのパッケージは同梱されているPKGBUILDというビルドファイルに基づいている。大抵の場合PKGBUILDにパッケージの対象アーキテクチャが書かれており、表記意外のアーキテクチャでは利用できない。<br>
ペンテスト環境を整える上で必須のMetasploitとJohnもどいう訳かこれに引っかかっている為、そのPKGBUILDを書き直すかyayを使ってAURのリポジトリから落とす。今回はyayで行く。Aarch64(スマートフォン)プロセッサだとBlackArchと本家で利用できないパッケージが多々存在する。その都度AURを使うといいだろう。<br>

![](https://raw.githubusercontent.com/Extr3m4R4710/zen-conf-record/main/IMG/23-11-19/blackarch-rootless-on-android/arch-x86-64-metasploit.png)

本稿で示した手筈の通り進めていればyayは利用可能。先程作成したユーザー名を`sudo -i -u <ここにmyuser>`という形で渡してやると非rootユーザーにログインできる。<br>
yayはrootユーザーでは利用できない。 <br>

ここでは2つの不足しているパッケージをインストールする。<br>

```bash
yay -S john-git metasploit-git --noconfirm 

```
AURパッケージの命名規則として`XXX-git`というものがあるが、これはgitサーバーのソースファイルを参照しているという表しで、基本的にアーキテクチャー指示も含まれていないのでスマートフォンにもインストールできる。<br>

他にインストールしたいツールがあるなら `yay -Ss`コマンドで巡回するといいだろう。<br>

`追記: 今確認したところAURのjohn-gitはビルド時にエラーでつまりトリップしてしまう為KaliのGitLabで開発されているjohnを自前でビルドする必要がある。`<br>
[https://gitlab.com/kalilinux/packages/john](https://gitlab.com/kalilinux/packages/john)<br>

## BlackArchのカスタム
ハッカーの好みとしてbashを使うのかzshを使うか、はたまたfishか、プロンプトテーマは自作するかという終わりのない悩みはあると思うが、概して嗜好なので好きなものを使っていただきたい。
BlackArchは`blackarch-config-bash`と`blackarch-config-zsh`というプロンプトテーマセットをpacmanで提供している。職人気質の殺風景は避けられるだろう。
参考に私が`blackarch-config-bash`をもとに作成したテーマを置いておく。<br>
```bash
# .bashrc
#
#
## v1.0.0
#
# colors
darkgrey="$(tput setaf 8)"
white="$(tput setaf 15)"
blue="$(tput setaf 12)"
cyan="$(tput setaf 14)"
red="$(tput setaf 9)"
nc="$(tput sgr0)"

# exports
export PATH="${HOME}/bin:/sbin:/usr/bin:/usr/sbin:/usr/local/bin:"
export PATH="${PATH}/usr/local/sbin:/opt/bin:/usr/bin/core_perl:/usr/games/bin:var/lib/snapd/snap/bin:"

if [[ $EUID -eq 0 ]]; then
  export PS1="\[$blue\][ \[$darkgrey\]\u@\[$cyan\]\H \[$darkgrey\]\w\[$darkgrey\] \[$blue\]]\\[$darkgrey\]# \[$nc\]"
else
  export PS1="\[$blue\][ \[$darkgrey\]\u@\[$cyan\]\H \[$darkgrey\]\w\[$darkgrey\] \[$blue\]]\\[$darkgrey\]% \[$nc\]"
fi

export LD_PRELOAD=""

# alias
alias ls="ls --color"
alias vi="vim"
alias sqlmap="sqlmap --tor"
alias shred="shred -uvzf"
alias nmap="nmap -vvv -Pn --dns-server 1.1.1.1"
alias wget="wget -U 'noleak'"
alias curl="curl --user-agent 'noleak'"

# source files
[ -r /usr/share/bash-completion/completions ] &&
  . /usr/share/bash-completion/completions/*


# custom functions
#

ops-mode()
{
  export PS1="\[$blue\]┌[\[$darkgrey\]\u@\[$cyan\]\H\[$blue]\]-[\[$cyan\]$(date +"%Y-%m-%d/%H:%M:%S")\[$blue\]]-[\[$cyan\]\w\[$blue\]]\n\[$blue\]└-\[$blue\]╼\[$cyan\]#>> \[$nc\]"
}

clwhois()
{
  curl -sL http://cli.fyi/$1 | jq .
}

remoteshell()
{
  sshpass -p 'segfault' ssh -D 1080 root@segfault.net
}

myip()
{
  curl -sL ip-api.com/json | jq .
}

wiki()
{
  case $1 in
    ja)
      proxychains4 -q w3m ja.wikipedia.org/wiki/$2
    ;;
    en)
      proxychains4 -q w3m en.wikipedia.org/wiki/$2
    ;;
    arch)
      proxychains4 -q w3m wiki.archlinux.org/title/$2
  esac

}

whatime()
{
  date +"%Y-%m-%d %H:%M"
}

```
一通り設定が終わると私の場合こうなる。<br>
![](https://raw.githubusercontent.com/Extr3m4R4710/zen-conf-record/main/IMG/23-11-19/blackarch-rootless-on-android/backarch-on-my-phone.jpg)

## 実際に動かしてみる
[nh紹介動画](https://www.youtube.com/watch?v=KxOGyuGq0Ts)でおなじみのNmapでdns.google.comを調べてみる、1つ注意点があり、ステルススキャンができない。Termux全般に言えることだが、端末自体はroot化されていない為、ステルススキャンを行う上で必要なSYNパケットを生成するソケットをNmapは使用できない。それどころかNmapは実行者がrootである事を検出すると自動でステルススキャン(-sS)を試みるがソケットを触れない以上Nmapは**rootユーザーでは使用できない**。そのため非rootユーザーでTCPスキャン(-sT)をを行いう必要がある。非rootユーザーのデフォルトは常に-sT <br>

![](https://raw.githubusercontent.com/Extr3m4R4710/zen-conf-record/main/IMG/23-11-19/blackarch-rootless-on-android/blackarch-nmap-TcP.jpg)

## 最後に
Nethunter Rootlessと同等のBlackArchを揃えることができた。しかしnhには`kex`と呼ばれるVNCサーバー機能があり、これによってグラフィカルなデスクトップ環境を提供している。<br>
次号はVNC機能構築についてもトライしたい。 <br>

## 余談 pacmanにおけるblackarchカテゴリを使うべきか

BlakArchの提供するツールはグールプ化されている。例えば`pacman -S blackarch-scanner`と叩くと即座に各種スキャナーをインストールすることができる。<br>
![](https://raw.githubusercontent.com/Extr3m4R4710/zen-conf-record/main/IMG/23-11-19/blackarch-rootless-on-android/blackarch-guide-group2.png)
![](https://raw.githubusercontent.com/Extr3m4R4710/zen-conf-record/main/IMG/23-11-19/blackarch-rootless-on-android/blackarch-guide-group.png)

しかし、基本的にAndroidはAarch64ファミリーのCPUで動くためx86系ではない。もし仮にグルーピングされたパッケージにx86_64用依存関係が含まれていた場合、正常なインストールができないので(依存関係を調べ上げ、かつAURかgitから落としてパスを通せば別)欲しいパッケージが手に入らなかったりする。例えばMetasploitはx86_64パッケージとして用意されてるし、BlackArch環境を作るぐらいなら、nhを使った方が間違いなく早い。しかしnhは数年に1回の頻度で際セットアップで難が出るので、臨時でなら使えるかもしれない。PKGBUILDのアーキテクチャ指定をどうするのかが悩みの種となるだろうが...<br>
