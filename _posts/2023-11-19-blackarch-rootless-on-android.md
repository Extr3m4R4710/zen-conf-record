---
layout: post
title: AndroidスマートフォンにBlackArchを構築する。root化不要
data: 2023-11-19 5:31 +0900
tag: [linux, chroot, モバイル, hack]
---
![](https://raw.githubusercontent.com/Extr3m4R4710/zen-conf-record/main/IMG/23-11-19/blackarch-rootless-on-android/blackarch-on-my-phone-with-keybord.jpg)

Androidスマートフォンをペンテスト仕様にカスタムする方法としてNethunterをROM焼きするか、TermuxにNethunter Rootless(nh)を導入する方法が一般的だが、Kali(特にnh rootless)はBlackArchと比較してツール数が少なかったり幅広い開発・侵入支援モジュールが不足している側面も見受けられる。ユニバーサルなDebianとハッカーフレンドリーなArchLinuxの違いだと考えている。ほんの出来心でBlackArch環境をTermux上に構築してみる。<br>

本稿ではNetHunter Rootlessの代替として機能するBlackArch環境構築の手引きとしてnh以外の選択肢を示すことにする。<br>

# 目次

[TOC]

先に自動化スクリプトを公開しておく。これをインストールしたProot ArchLinux環境で実行。<br>

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

#先に入手できる範囲のツールを取得、arch=('x86_64')で取得できない場合はyayで補う
pacman -Syyu && pacman -S nmap sqlmap hydra hashcat recon-ng sn0int sherlock
```
## Termuxの取得
[GitHub](https://github.com/termux/termux-app/releases)でAPKを入手するか、[F-Droid](https://f-droid.org/)から入手できる。Google PlayのTermuxは4年ほど前から更新されていない。<br>

## Prootの準備
pkgコマンドでパッケージのアップグレードとproot-distroをインストールする。確認を求められるのが嫌なら各々`-y`オプションを付ける。 <br>
```bash
pkg upgrade
pkg install proot-distro
```
セットアップが完了したら`proot-distro install archlinux`を実行しルートディレクトリとなる圧縮ファイルをダウンロードする。prootはchrootのtermux実装でchrootと同じくルートディレクトリ構造をファイルシステムと認識し、その中で仮想的なOSとして振る舞う。その為xz形式(だっけ？)を展開する。<br>

ArchLinuxのインストールが完了すれば`proot-distro login archlinux`でログイン。その後先程のシェルスクリプトを実行する。<br>
```bash
# スクリプトとして実行する場合。ファイルに落とし込む場合はCUIエディタを使うか
# Prootファイルシステムの/rootディレクトリに直接置く
chmod +x <任意のファイル名>.sh
```

blackarchリポジトリにアクセスできない場合、`/etc/pacman.d/blackarch-mirrorlist`を適宜修正する。例えばKDDIの日本サーバーのコメントアウトを外すなど。<br>

## ユーザーの作成
rootでは動かないyayやnmapを使うため(後述)、非rootユーザーを作成する。その為のbase-devel。 <br>
### 手順
まずホームディレクトリとログインシェル、同時にユーザーを設定する。 <br>
```bash
useradd -m -s /bin/bash username # ユーザー名とシェルは好みに合わせて変更すること
```

一旦rootに戻どりパスワードを設定する。 <br>
```bash
passwd username
```

## PKGBUILD上の制約でインストールできないツールをyayで補う

ArchLinuxのパッケージは同梱されているPKGBUILDというビルドファイルに基づいている。大抵の場合PKGBUILDにパッケージの対象アーキテクチャが書かれており、表記以外のアーキテクチャでは利用できない。<br>

ペンテスト環境を整える上で必須のMetasploitとJohnもどういう訳かこれに引っかかっている為、そのPKGBUILDを書き直すかyayを使ってAURのリポジトリから落とす。今回はyayで行く。Aarch64(スマートフォン)プロセッサだとBlackArchと本家(他Arch系)で利用できないパッケージが多々存在する。動作に問題がない(Metasploitのようにスクリプト言語で書かれているなど)場合、その都度AURを使うといいだろう。<br>

![](https://raw.githubusercontent.com/Extr3m4R4710/zen-conf-record/main/IMG/23-11-19/blackarch-rootless-on-android/arch-x86-64-metasploit.png)

本稿の手順の通り進めていればyayは利用可能。先程作成したユーザー名を`sudo -i -u <ここにmyuser>`という形で渡してやると非rootユーザーにログインできる。<br>
yayはrootユーザーでは利用できない。 <br>

ここでは2つの不足しているパッケージをインストールする。<br>

```bash
yay -S john-git metasploit-git --noconfirm

```
AURパッケージの命名規則として`XXX-git`というものがあるが、これはgitリポジトリのソースファイルを参照しているという表しで、基本的にアーキテクチャー指示は含まれていないのでスマートフォンにもインストールできる。<br>

他にインストールしたいツールがあるなら `yay -Ss`コマンドで巡回するといいだろう。<br>

`追記: 今確認したところAURのjohn-gitはビルド時のエラーでトリップしてしまう為、KaliのGitLabで保守されているjohnを自前でビルドする必要がある。`<br>
[https://gitlab.com/kalilinux/packages/john](https://gitlab.com/kalilinux/packages/john)<br>

## BlackArchのカスタム
ハッカーの好みは一概ではない。bashを使うのかzshを使うか、はたまたfishか、プロンプトテーマは自作するかという終わりのない悩みはあると思うが、概して嗜好なので好きなものを使っていただきたい。BlackArchは`blackarch-config-bash`と`blackarch-config-zsh`というプロンプトテーマをpacmanで提供している。職人気質の殺風景は避けられるだろう。<br>

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
一通り設定が終わるとこうなる。<br>
![](https://raw.githubusercontent.com/Extr3m4R4710/zen-conf-record/main/IMG/23-11-19/blackarch-rootless-on-android/backarch-on-my-phone.jpg)

## 実際に動かしてみる
[nh紹介動画](https://www.youtube.com/watch?v=KxOGyuGq0Ts)でおなじみのNmapでdns.google.comを調べてみる、1つ注意点があり、ステルススキャンができない。Termux全般に言えることだが、端末自体はroot化されていない為、ステルススキャンを行う上で必要なSYNパケットを生成するソケットをNmapは使用できない。それどころかNmapは実行者がrootである事を検出すると自動でステルススキャン(-sS)を試み、それでいてソケットを触れない為、結局Nmapを**使用できない**。そのため非rootユーザーでTCPスキャン(-sT)を行う必要がある。非rootユーザーのデフォルトは常に-sT <br>

![](https://raw.githubusercontent.com/Extr3m4R4710/zen-conf-record/main/IMG/23-11-19/blackarch-rootless-on-android/blackarch-nmap-TcP.jpg)

## 最後に
Nethunter Rootlessと同等のBlackArchを揃えることができた。nhには`kex`と呼ばれるVNCサーバー機能があり、これによってグラフィカルなデスクトップ環境を提供している他、環境の起動時にrootを簡単に指示できる(nh -r)。またコマンドを渡すことで(nh sqlmap -u "https://YYY.XXXXXX.xyz")NetHunter環境のツールを直接走らせるというDockerコンテナのような扱い方ができる。同様にproot-distroでも`proot-distro login archlinux -- /usr/bin/sqlmap -u "https://YYY.XXXXXX.xyz"`という具合に使えはするものの、nhと比べて若干取り回しが悪い。nhは`-r`という短い指示でrootを呼び出せる事に対して、proot-distroは頑張っても`pd login --user root archlinux`となってしまい、簡便さに劣ってしまう。このような直接起動は目的とツールが明確な場合好んで使われる、わざわざ仮想環境のシェルを立ち上げる必要がないので、うまく言えないがサイバーセキュリティにおけるのタクティカル的なアドバンテージがある。細部に凝った設計はNetHunterの強みと言える。<br>

おまけ:nethunter(nh)で利用可能なオプション

| Command | to |
|:-------:|:----:|
| nethunter(nh) | start Kali NetHunter command line interface |
| nethunter kex passwd(nh kex passwd) | configure the KeX password (only needed before 1st use) |
| nethunter kex &(nh kex &) | start Kali NetHunter Desktop Experience user sessions |
| nethunter kex stop(nh kex stop) | stop Kali NetHunter Desktop Experience |
| nethunter <command>(nh <command>) | run in NetHunter environment |
| nethunter -r(nh -r) | start Kali NetHunter cli as root |
| nethunter -r kex passwd(nh -r kex passwd) | configure the KeX password for root |
| nethunter -r kex &(nh -r kex &) | start Kali NetHunter Desktop Experience as root |
| nethunter -r kex stop(nh -r kex stop) | stop Kali NetHunter Desktop Experience root sessions |
| nethunter -r kex kill(nh -r kex kill) | Kill all KeX sessions |
| nethunter -r <command>(nh -r <command>) | 	run <command> in NetHunter environment as root |

次号はProot BlackArchのVNC機能構築についてもトライしたい。 <br>

## 余談: 若干のblackarchカテゴリの説明とBlackArch(ARM)環境の所感

BlackArchが提供するツールはグールプ化されている。例えば`pacman -S blackarch-scanner`と叩くと即座に各種スキャナーをインストールすることができる。<br>
![](https://raw.githubusercontent.com/Extr3m4R4710/zen-conf-record/main/IMG/23-11-19/blackarch-rootless-on-android/blackarch-guide-group2.png)
![](https://raw.githubusercontent.com/Extr3m4R4710/zen-conf-record/main/IMG/23-11-19/blackarch-rootless-on-android/blackarch-guide-group.png)だが、基本的にAndroidはARM(Aarch64)ファミリーのCPUで動く為、x86_64パッケージは利用できない。もし仮にグルーピングされたパッケージにx86_64が系が含まれていた場合、インストールできない(pacmanが除外してくる)ので欲しいパッケージが手に入らなかったりする(AURからインストールするか、gitで落としてパスを通すか、他PKGBUILDを編集すれば別)。<br>

例えばBlackArch(正確にはArchLinuxのExtra)で提供されているMetasploitはx86_64パッケージとして用意されてるし、BlackArch環境を作るぐらいなら、初めからnhを使った方が間違いなく早く、そして手堅いと思う。それでも使えるツールの幅は自体はKali(nh)より広く、Kali(nh)では提供されていないツールを手軽にpacmanから選べるという最大のメリットがある。<br>

nhに限らずKaliの一番の不満はツールが古かったりする他、調査やペンテスト演習で一々不足分を揃える必要があり、GitHubを何回も巡るハメになるという絶望的なめんどくささにある。先程述べた通りAarch64環境でBlackArchを用意する場合、なぜか鉄板パッケージがx86_64にカテゴリされているなどの不満はあるが、パッケージマネージャで賄えるツールの幅は本当に広く、ツールコレクションとしてはKali(nh)より優れている。<br>

確かに問題のパッケージを揃える過程でPKGBUILDの`arch=('x86_64')`都度編集してmakepkgでビルドしたり、AURやgitを使うなど骨が折れる、ただnhが壊れてしまってなおかつクリーンインストールができない(Androidのバージョンが更新される度にnhのクリーンインストールで積む)、場合やそもそもKaliLinux自体に変えられない不満があるなら試してみる価値はあると思う。CPUアーキテクチャ的に問題ないが`arch=('x86_64')`であるためインストールできないというパッケージは、ARM BlackArchを構築して実際に使用してみた限りそう多くない。しっかりリサーチとリストアップを行い、不足分の補填を自動化するスクリプトを作成すれば、そう悩ましい問題でもないと思う。<br>

x86_64マシンの場合、KaliLinuxで利用できるツールの数は`約600`と幾らかであるのに対して、BlackArch(x86_64)で利用できるツールの総量は公式サイトの記述よると`2845点`に上る。単純計算だけで見てもKaliLinuxの`600点と比較して2245点もの差`を付けている。この差はPKGBUILDの`arch=('x86_64')`から来るハンデを持つBlackArch(ARM)とKaliLinux(nh)を比較しても殆ど変わらないと思う。実際、[BlackArchの公式GitHubリポジトリで収録パッケージのPKGBUILDを閲覧することができるが、arch=('x86_64')と明示されたPKGBUILDは僅か134点](https://github.com/search?q=repo%3ABlackArch%2Fblackarch+arch%3D%28%27x86_64%27%29&type=code)に留まっている。そして現在、BlackArchコミュニティが保守・管理するPKGBUILDは[総数約4000点(4.2k)となっている。](https://github.com/search?q=repo%3ABlackArch/blackarch%20path%3APKGBUILD&type=code)<br>

細部に凝った設計と鉄板ツールが簡単に利用できるnhと驚異的なツール数を誇るBlackArch、どちらかを使うかは使用者の好みと目的に左右される節は大きい。私は長らくBlackArchの個人的ファンなので、少しでも日本でBlackArchの注目が集まって欲しいばかりにこの記事を書いてみた。間接的にでも、この記事がBlackArchコミュニティへの貢献になれば幸い。
