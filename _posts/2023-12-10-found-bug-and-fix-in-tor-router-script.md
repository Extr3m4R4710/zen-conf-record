---
layout: post
title: "[書きかけ] 匿名通信用ラッパーtor-routerのフォークを書いた話"
data:  2024-01-04 +0900
tag: [linux, tor, 匿名通信, hack, 開発]
---
多忙によりtor-routerの修正とNetSpectreの公開についてまとめた記事が書けませんでしたが、時間が確保できましたので公開しておきます。
なおこの記事は書きかけの項目ですので、その点をご了承願います。

# tor-routerについて
tor-routerはEdu4rdSHL氏により作成された簡素な匿名通信ラッパーです。iptablesの制御による透過プロキシとしてTorによるネットワーク通信を強制するシェルスクリプトとして動作します。

# なぜフォークを書いたのか
そのtor-routerにバグがあり、特定のウェブサイトでIPリークが起こる問題がありました。
そのためフォークする形でバグを修正し、一旦ソースコードの研究も兼ねてそれをベースとして独自に改良する形をとり、得られた知見を元に一段落してから本家へプルリクを投げました。それらは後述します。

# バグに気づいた経緯
私はVPNサーバーがダウンしている間はレポジトリから提供されているtor-routerを使ってコンピュータの全通信をTorで難読化していました。その際[Qiitaに掲載されている](https://qiita.com/keiya/items/589df899ffd167f4c909)torrcのカスタムを試した後、最終チェックとして[ipinfo.io](https://ipinfo.io)で回線を確認したところプロバイダーのIPアドレスが漏洩していました。私が使っているVPNサービスはサーバーが国家支援型グループ等から定期的にDDoS攻撃を受けている為、全面的にダウンしている場合他に宛てがないのでTorを使う事にしています。そしてtor-routerのバグに気づいたその時を同じくし該当VPNサーバーの方も全面的にダウンしてしまった為、オリジナル作者の修正を待っている余裕が無い事もあり、自前でフォークとして修正した後、得られた成果をプルリクとして投げることにしました。

![](https://raw.githubusercontent.com/Extr3m4R4710/zen-conf-record/main/IMG/24-01-04/my-post-about-tor-router-bug.png)

# tor-routerの改良
## 関数を用いて整形
tor-routerのソースコードを参照して分かりましたが、コード全体が荒削りでした。例えばこのrestart引数は正常に動きません。

![](https://raw.githubusercontent.com/Extr3m4R4710/zen-conf-record/main/IMG/24-01-04/restart-arg-in-bad-case.png)

restart引数の問題は恐らくオリジナル作者が`case`文の引数をそのままコマンドと誤認していることが原因です。原型を留めて修正するなら以下のようにbashの特殊変数を使い文中でtor-routerそのものを指し示すべきです。
```bash
restart)
    $0 stop
    sleep 1
    $0 start
;;
```
![](https://raw.githubusercontent.com/Extr3m4R4710/zen-conf-record/main/IMG/24-01-04/functionality.png)

今回は可読性と開発効率を上げるため関数化しました。またセキュリティ強化の為デフォルトでipv6接続を無効化しておきます(スクリプト適用時のみ)。
なおこの時点でフォークとして再設計していた為、本稿で改良しているスクリプトはフォーク元と区別するため`NetSpectre`と新たに名付け公開しました。

## スプリットトンネルの改良
![](https://raw.githubusercontent.com/Extr3m4R4710/zen-conf-record/main/IMG/24-01-04/subnet.png)

tor-routerではTor経由でルーティングしない宛先として`192.168.1.0/24`と`192.168.0.0/24`というローカルネットワークが明示的にルーティングから除外されています。

![](https://raw.githubusercontent.com/Extr3m4R4710/zen-conf-record/main/IMG/24-01-04/subnet-netspectre.png)

ここも少し手を加えました。より一般的なアドレスブロックである`192.168.0.0/16` `172.16.0.0/12 ` `10.0.0.0/8`に置き換えました。理由は大規模ネットワークでの運用も視野に入れた為です。ipcalcで計算してみると分かりやすいと思います。
```bash
 ~>>> ipcalc 192.168.0.0/16
Address:   192.168.0.0          11000000.10101000. 00000000.00000000
Netmask:   255.255.0.0 = 16     11111111.11111111. 00000000.00000000
Wildcard:  0.0.255.255          00000000.00000000. 11111111.11111111
=>
Network:   192.168.0.0/16       11000000.10101000. 00000000.00000000
HostMin:   192.168.0.1          11000000.10101000. 00000000.00000001
HostMax:   192.168.255.254      11000000.10101000. 11111111.11111110
Broadcast: 192.168.255.255      11000000.10101000. 11111111.11111111
Hosts/Net: 65534                 Class C, Private Internet

 ~>>> ipcalc 192.168.0.0/24
Address:   192.168.0.0          11000000.10101000.00000000. 00000000
Netmask:   255.255.255.0 = 24   11111111.11111111.11111111. 00000000
Wildcard:  0.0.0.255            00000000.00000000.00000000. 11111111
=>
Network:   192.168.0.0/24       11000000.10101000.00000000. 00000000
HostMin:   192.168.0.1          11000000.10101000.00000000. 00000001
HostMax:   192.168.0.254        11000000.10101000.00000000. 11111110
Broadcast: 192.168.0.255        11000000.10101000.00000000. 11111111
Hosts/Net: 254                   Class C, Private Internet

```
## Iptablesの再設計
一番苦労した箇所です。具体的には[ParrotSec/anonsurf](https://github.com/ParrotSec/anonsurf)と[iptablesチュートリアル](https://straypenguin.winfield-net.com/ipttut/output/index.html)を参考にしました。また日本語の[manpage](https://ja.manpages.org/iptables/8)も非常に役立ちました。

tor-routeのソースと採用されているIptablesを交えて見ていきます。なおコードは[NetSpectreが直接のベース元としているtor-routerのコミット(290d0b1)](https://github.com/Edu4rdSHL/tor-router/blob/290d0b1e29a13a4e1d4f109a6a31bdd1da523dc9/files/tor-router)です。

このプログラムではIptablesをファイアウォールとしてではなく、Torトランスポートを利用した透過的ローカルルーターとして、全てのネットワーク通信でTorSocksを使用し経由させる役割を果たします。例えばRasberry Piを換装した[Onion Pi AP](https://raspberrypi-japan.com/router-tor-raspberry-pi-5/)はWiFi接続したOnion Piを経由することによりOnion Piから出る通信にTor回線の経由を強制させることが出来ます。しかしtor-routerやそれをベースとするNetSpectreはホストマシンのIptablesを使い透過的かつ仮想的な匿名化ルーターを提供している為、物理的な装置を必要としません。当然ながらWindowsの全ネットワーク通信をTorでカバーする場合はOnion Piかそれと同等のアクセスポイントを使用するべきです(筆者はWindows環境におけるフルデバイスTorの手法をあまり知らない)。

より詳しいTorトランスポートに関する説明はArchWikiかTorのWikiをご覧ください。
[https://wiki.archlinux.org/title/tor#Transparent_Torification](https://wiki.archlinux.org/title/tor#Transparent_Torification)
[https://gitlab.torproject.org/legacy/trac/-/wikis/doc/TransparentProxy](https://gitlab.torproject.org/legacy/trac/-/wikis/doc/TransparentProxy)

しかしながら、一般的な方法ではないので、読み解く前段階として短い説明を行います。
### tor-routerのソースコード
先にtor-routerのIptablesについて軽く触れておきます
```bash
#!/bin/bash

RULES="/var/tmp/tor-router.save"

# Tor's TransPort
TRANS_PORT="9040"

case "$1" in
start)
	if test -f "$RULES"; then
		echo "$RULES exists. Either delete it, or stop tor-router first."
		exit 1
	else
		iptables-save >$RULES
		# Executable file to create rules for transparent proxy
		# Destinations you do not want routed through Tor
		NON_TOR="192.168.1.0/24 192.168.0.0/24"
		# the UID Tor runs as, actually only support for Debian, ArchLinux and Fedora as been added.
		if command -v pacman >/dev/null; then
			TOR_UID=$(id -u tor)
		elif command -v apt >/dev/null; then
			TOR_UID=$(id -u debian-tor)
		elif command -v dnf >/dev/null; then
			TOR_UID=$(id -u toranon)
		else
			echo "Unknown distro, please create report the issue to https://github.com/edu4rdshl/tor-router/issues"
			exit 1
		fi

		if ! command -v tor >/dev/null; then
			echo "You need to install the tor package."
			exit 1
		elif ! systemctl is-active tor.service >/dev/null; then
			echo "The tor service is not active, please start the tor service before running the script."
			exit 1
		elif ! command -v iptables >/dev/null; then
			echo "You need to install the iptables package."
			exit 1
		else
			iptables -F
			iptables -t nat -F
			iptables -t nat -A OUTPUT -m owner --uid-owner "$TOR_UID" -j RETURN
			iptables -t nat -A OUTPUT -p udp --dport 53 -j REDIRECT --to-ports 5353

			for NET in $NON_TOR 127.0.0.0/9 127.128.0.0/10; do
				iptables -t nat -A OUTPUT -d "$NET" -j RETURN
			done

			iptables -t nat -A OUTPUT -p tcp --syn -j REDIRECT --to-ports $TRANS_PORT
			iptables -A OUTPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

			for NET in $NON_TOR 127.0.0.0/8; do
				iptables -A OUTPUT -d "$NET" -j ACCEPT
			done

			iptables -A OUTPUT -m owner --uid-owner "$TOR_UID" -j ACCEPT
			iptables -A OUTPUT -j ACCEPT
		fi
	fi
	;;
stop)
	if test -f "$RULES"; then
		echo "Restoring previous rules from $RULES"
		iptables -t nat -F
		iptables-restore <"$RULES"
		rm "$RULES"
	else
		echo "$RULES does not exist. Not doing anything."
		exit
	fi
	;;
restart)
	stop
	sleep 2
	start
	;;
*)
	echo "Usage: $0 {start|stop|restart}"
	;;
esac

```

### テーブル・チェイン・ターゲットについて
Iptablesの各種テーブル・チェイン・ターゲットについて軽く触れていきます。初めてIptablesを目にする方もいると思うので概説に限りtor-routerで採用されているIptables以外にも踏み込んで解説します。

#### テーブル・チェイン・ターゲットの概説
テーブルは`iptables -t <tables ...>`の形で指定され各チェインは`-A`オプションでルールが追加され、`-j `でターゲットへジャンプします。例えばOUTPUT(出て行くパケット)に対する任意の条件でマッチしたパケットは、その後`-j`の次に指定されるターゲットが行く末を握っています。例として任意の`条件A`にマッチしたパケットが`ACCEPT`に出会った時そのまま通過・送信され、仮に`DROP`であった場合、通過は阻まれ送信されません。受け取った瞬間床に投げ捨てるイメージです。

`RETURN`は、そのチェインの検討を中止、または、親チェイン内の次のルールから評価を再開する事を示すターゲットです。組み込み済みチェイン(例えばINPUT)に追加されたルールに`RETURN`が使用されている場合、パケットはデフォルトポリシーによって処理されます。尚、iptablesのデフォルトポリシーは特にカスタムされている例外を除き全て`ACCEPT`です。この場合パケットはデフォルトポリシーによって通過します。
![](https://raw.githubusercontent.com/Extr3m4R4710/zen-conf-record/main/IMG/24-01-04/iptables-default-policy.png)


```bash
# 例1: OUTPUTチェイン
 iptables  -A OUTPUT < ...条件... > -j ACCEPT
 iptables  -A OUTPUT < ...条件... > -j DROP
# 例2: 拡張ターゲット 。マッチしたパケットをカーネルログに記録する。主にファイアウォール(INPUTチェインと合わせて)で使われる
  iptables  -A OUTPUT < ...条件... > -j LOG
```
#### テーブルとチェインの説明
採用されているテーブルは以下のとおりです

- filter: `-t`オプションが渡されていなければこのテーブルがデフォルトとです。これには以下のチェインが含まれます
  - `INPUT `(入ってくるパケットに対するチェイン)
  - `FORWARD `(当該コンピュータを経由するパケットに対するチェイン)
  - `OUTPUT` (ローカルで生成されたパケットに対するチェイン)

- nat: パケットの送信元または宛先フィールドを変更する目的にだけ使用するテーブルです。このテーブルの内今回使われるチェインは以下のとおりです。
   -  `OUTPUT`(filterのOUTPUTとは少し違い、ローカルで生成されたパケットをルーティングの前に変換するチェイン)

#### ターゲット
Iptablesにおいてルールとはパケットを判断する基準(条件)と条件にマッチした場合ジャンプするターゲットで構成されています。このコードではこのようなターゲットが使われています。

- `RETUEN`
- `REDIRECT`
- `ACCEPT`

```bash
            iptables -F
            iptables -t nat -F
            iptables -t nat -A OUTPUT -m owner --uid-owner "$TOR_UID" -j RETURN
            iptables -t nat -A OUTPUT -p udp --dport 53 -j REDIRECT --to-ports 5353

            for NET in $NON_TOR 127.0.0.0/9 127.128.0.0/10; do
                iptables -t nat -A OUTPUT -d "$NET" -j RETURN
            done

            iptables -t nat -A OUTPUT -p tcp --syn -j REDIRECT --to-ports $TRANS_PORT
            iptables -A OUTPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

            for NET in $NON_TOR 127.0.0.0/8; do
                iptables -A OUTPUT -d "$NET" -j ACCEPT
            done

            iptables -A OUTPUT -m owner --uid-owner "$TOR_UID" -j ACCEPT
            iptables -A OUTPUT -j ACCEPT

```
tor-routerは`iptables-save >$RULES`(注: $RULESは/var/tmp/tor-router.saveを指している)で現時点のIptablesの内容をバックアップした後`iptables -F` `iptables -t nat -F`コマンドを実行しIptableの`filter`テーブルと`nat`を初期化しています。

![](https://raw.githubusercontent.com/Extr3m4R4710/zen-conf-record/main/IMG/24-01-04/set-nat-and-dns-redirect.png)

段階を分けて見ていきます。まず`iptables -t nat -A OUTPUT -m owner --uid-owner "$TOR_UID" -j RETURN`でiptablesは`nat`テーブルの`OUTPUT`チェインによりTorプロセスが生成するパケットに変換されている場合`RETURN`(評価終了)し、また`iptables -t nat -A OUTPUT -p udp --dport 53 -j REDIRECT --to-ports 5353`でDNSトラフィックをlocalhost:5353へ文字通り`リダイレクト`しています。

ルールの中でRETURNにジャンプしたパケットは現在実行しているルールの評価を停止し、この場合`-t nat OUTPUT`チェーンへ評価を差し戻します(突き落とす)。差し戻されたパケットの処遇はそのチェーンに採用されているデフォルトポリシーによって決定されます。

 なおREDIRECTはnatテーブル内のチェインでのみ有効でこのターゲットにジャンプするとパケット送信アドレスをコンピュータ自身のIPアドレスに変換します(所謂localhost)
```
REDIRECT
このターゲットは、 nat テーブル内の PREROUTING チェイン及び OUTPUT チェイン、そしてこれらチェインから呼び出される ユーザー定義チェインでのみ有効である。 このターゲットはパケットの送信先 IP アドレスを マシン自身の IP アドレスに変換する。 (ローカルで生成されたパケットは、アドレス 127.0.0.1 にマップされる)。
```

