# 障害・エラー記録

起きたエラーや障害は、解決したら必ずここに追記する。「訓練」と「実際に起きた障害」に分け、それぞれ新しいものを上に書く。

## 書式

```
### YYYY-MM-DD タイトル(例:Nginx再起動後に502)
- 環境:手元VM(AlmaLinux) / EC2 など
- 症状:何が起きたか、エラーメッセージ
- 切り分け:何を順に確認したか(実行したコマンドと結果)
- 原因:
- 対処:
- 再発防止・学び:
```

---

<!-- ここから下に追記 -->

## 障害対応の訓練(Step 3、手元の VM でわざと起こしたもの)

### 2026-10-30 【訓練e】VM の /app/ で 502 Bad Gateway
- 環境: 手元VM(AlmaLinux 9)
- 症状: `curl -I http://127.0.0.1/app/` が `502 Bad Gateway`。`error.log` に `connect() to 127.0.0.1:3000 failed (13: Permission denied)`。ファイルではなく接続が拒否されている
- 切り分け: `systemctl status nginx`(active)→ `curl -I` で `/` は 200、`/app/` は 502 → `sudo cat /var/log/nginx/error.log` → 502 の直後に `sudo ausearch -m avc -ts recent` で `denied { name_connect }` を確認
- 原因: SELinux が、nginx(`httpd_t`)から 3000 番ポート(`ntop_port_t`)への TCP 接続(`name_connect`)を拒否した(`httpd_can_network_connect` が off になっていた)
- 対処:
    - `sudo setsebool -P httpd_can_network_connect on` で許可
    - `getsebool httpd_can_network_connect` で on を確認
    - 502 が出ていたのと同じ `curl -I .../app/` で 200 を確認
- 学び:
    - 前日(10/29)、訓練の準備で VM に `/app/` の中継を足したときにも同じ 502 が起きていた。そのときは nginx を再起動してから `ausearch` して記録が出ず(誰も 3000 番に接続しないため)、確認も `/`(静的ページ)でしてしまった。今回はそのつまずきを活かし、502 の直後に `ausearch` して一度で見つけ、`/app/` で 200 を確認した
    - 同じ 502 でも、error.log の数字で見分けられる。`111: Connection refused` はアプリが待ち受けていない(訓練d)、`13: Permission denied` は接続を止められている(SELinux)
    - `getsebool` は見るだけ(sudo 不要、`-P` は付けられない)。`setsebool` は設定を変える(sudo 必要)。boolean の名前は `getsebool -a | grep httpd` で探せる

### 2026-10-30 【訓練a】nginx が停止していた
- 環境: 手元VM(AlmaLinux 9)
- 症状: Web サイト(`/` も `/app/` も)が表示されない。nginx が止まっていた
- 切り分け:
    - `systemctl status nginx` で状況を確認 → `inactive (dead)` になっていた
    - `sudo journalctl -u nginx` で止まった時刻を確認 → 15:09:48 に `Deactivated successfully`(正常に停止)。そのあと Starting が無い
- 原因: 誰かが意図的に nginx を止めていた(`systemctl stop`)
- 対処: `sudo systemctl start nginx` → `active (running)` とサイトの表示を確認(止まっていたのは約 6 分)
- 再発防止・学び:
    - `dead` は意図的に止められた、`failed` はエラーなどが起きた
    - 誰が止めたかは、journalctl で止まった時刻を調べ、その時刻に sudo を使ったユーザーを `/var/log/secure` で特定する

### 2026-10-30 【訓練c】nginx が設定ミスで起動できない
- 環境: 手元VM(AlmaLinux 9)
- 症状: サイト(`/` も `/app/` も)にアクセスしても何も表示されない。`systemctl status nginx` が `failed`
- 切り分け:
    - `systemctl status hello-app` と `systemctl status nginx` で、動いているかをまず確認 → nginx が落ちていた
    - `sudo nginx -t` で設定を確認 → `[emerg] unknown directive "listne" in /etc/nginx/nginx.conf:40`
- 原因: nginx.conf の 40 行目に、`listen` のスペルミス(`listne`)があり、restart で起動できなくなった
- 対処: `sudo vi /etc/nginx/nginx.conf` で 40 行目を直す → `sudo nginx -t` で successful → `sudo systemctl start nginx`(止まっているので reload ではなく start)
- 再発防止・学び:
    - nginx の設定ファイルを直しただけでは、Web サイトは立ち上がらない。サービスが起動しているかを再確認し、接続する人と同じ立場から接続して確認する
    - `nginx -t` は、ファイル名と行番号まで教えてくれる

### 2026-10-30 【訓練d】 /app/ だけ nginx のエラーページ(502)
- 環境: 手元VM(AlmaLinux 9)
- 症状: Windows のブラウザで `/` は表示されるが、`/app/` だけ nginx のエラーページ(502)。nginx は active。hello-app は `activating (auto-restart)` を繰り返し、`curl 127.0.0.1:3000` は `Connection refused`
- 切り分け:
    - まずサービスが動いているか確認(`systemctl status hello-app`)
    - ユニットファイルを確認しに行く(`cat /etc/systemd/system/hello-app.service`)→ 設定は変わっていない
    - ログを確認(`journalctl -u hello-app -n 20`)→ `Error: EACCES: permission denied, open '/opt/hello-app/server.js'`
    - 権限を確認(`ls -l /opt/hello-app/server.js`)→ `-rw------- root root`
- 原因: `server.js` が 600(root だけ読める)になっていて、`User=hello` で動く node がファイルを開けずに、起動してはすぐ落ちていた
- 対処: `sudo chmod 644 /opt/hello-app/server.js`(全員が読める、書けるのは root のみ。元の状態に戻した)。`Restart=always` で自動的に立ち上がり、`active (running)` を確認
- 再発防止・学び: きれいにコマンドを使い分けて、原因を切り分けられるようになってきた。nginx とサービス(hello-app)の関係性を学んだ

### 2026-10-29 【訓練b】ブラウザでタイムアウト
- 環境: 手元VM(AlmaLinux 9)
- 症状: Windows のブラウザで / も /app/ もタイムアウト。SSH(22番)は普通につながる
- 原因: firewalld の services から http が消えていて、80番宛てのパケットが黙って捨てられていた
- 対処: `sudo firewall-cmd --permanent --add-service=http` → `sudo firewall-cmd --reload`。`--list-all` で `services: http ssh` を確認し、ブラウザで表示を確認
- 学び:
    - タイムアウトは「途中で黙って捨てられている」サイン。22番は通るのに80番だけダメなら、ポート単位で止める層(firewalld)を疑う
    - `--permanent` は設定ファイルに書くだけで、`--reload` しないと反映されない。今回は reboot で反映したが、本番ではサービスが止まるので reload を使う
    - 直すときは元の状態に戻すだけにする。https まで足してしまい、あとで外した
    - firewalld の設定は、見るだけでも sudo が必要

## 実際に起きた障害

### 2026-10-28 Nginx のリバースプロキシを設定したら 502 Bad Gateway
- 環境: EC2(AlmaLinux 9、SELinux Enforcing)
- 症状: Nginx 経由で `curl` すると `502 Bad Gateway`。`/var/log/nginx/error.log` に `connect() to 127.0.0.1:3000 failed (13: Permission denied)`
- 切り分け:
    - 502 → Nginx は動いていて、裏の Next.js から返事をもらえていないと判断
    - EC2 の中から `curl 127.0.0.1:3000` で Next.js に直接聞く → 返事あり。アプリは正常
    - `error.log` で `(13: Permission denied)`。ファイルではなく **ネットワークの接続**が拒否されている。同じサーバー内の接続なので firewalld ではない → SELinux を疑う
    - `sudo ausearch -m avc -ts recent` → `<no matches>`。`-ts recent` は直近 10 分だけで、502 が出たのは 20 分以上前だった → もう一度 502 を起こしてから検索
    - `denied { name_connect }`、`scontext=...httpd_t`、`tcontext=...ntop_port_t`、`permissive=0` を確認。`audit2why` で boolean `httpd_can_network_connect` の提案
- 原因: SELinux が、Nginx(`httpd_t`)から 3000 番ポート(`ntop_port_t`)への TCP 接続(`name_connect`)を拒否していた。Web サーバーが外へ接続することは、既定では許可されていない
- 対処: `sudo setsebool -P httpd_can_network_connect 1`(`-P` を付けて再起動後も残す)。`getsebool` で `on` を確認し、`curl` で 200 を確認
- 再発防止・学び:
    - 接続先が 3000 番で、ポートのラベルは `ntop_port_t` で管理されていた。Web サーバーが Web サーバー用ではないポートに(設定どおり)接続しに行ったが、既定では許可されていないので止まった
    - boolean で `httpd_can_network_connect` を on にして、Web サーバーがネットワークに接続してもよいかのスイッチをオンにした
    - `Permission denied` で、ファイルの権限で説明がつかないときは、SELinux を `ausearch -m avc` で見る
    - SELinux は切らずに、最小限のことだけを許可する。切るとアクセス制御ができなくなってしまう

### 2026-10-26 書き込めなかったユニットファイルに chmod a+rwx してしまった
- 環境: 手元VM(AlmaLinux 9)
- 症状: `/etc/systemd/system/hello-app.service` を編集しようとしたら書き込めなかったので、`sudo chmod a+rwx` した
- 切り分け: `history | grep chmod` で、いつ・どのファイルに何をしたかを特定した
- 原因: ファイルの持ち主が root だったので、一般ユーザーは読み込みしかできなかった
- 対処: `sudo chmod 644 /etc/systemd/system/hello-app.service` で元に戻し、`ls -l` で `-rw-r--r-- root root` を確認した
- 再発防止・学び: 書き込めないと言われたら sudo コマンドを使い書き込む。a+rwx のままだと一般ユーザーに不正アクセスなどされた際に任意のコードを実行できてセキュリティホールになってる、一般ユーザーがrootの権限を取れる状態だった。

### 2026-10-23 sshd設定を変えたのにパスワードログイン出来てしまう
- 環境: 手元VM(AlmaLinux 9)
- 症状: PasswordAuthentication noにしたのに、ssh root@...でパスワードを聞かれた
- 切り分け: `sudo systemctl reload sshd`を`reroad`と打ち間違えて、reloadされていなかった
- 対処: `sudo systemctl reload sshd`を実行し、新しいウィンドウで接続を確認した
- 再発防止: reloadの後はsystemctl statusでReloadedの記録を確認する、sshd -Tはファイルの値しか分からない。実際に接続して確かめる

### 2026-10-23 VMwareのホスト仮想アダプタ(VMnet8)が作れない
- 環境: Windowsホスト(VMware Workstation 26H1)
- 症状: 仮想ネットワークエディタで[ホスト接続]が有効にならない。[デフォルトに戻す]もタイムアウトする
- 切り分け:
    - 再起動しても同じ
    - VirtualBoxを疑って削除 > 削除自体が失敗(1603) `netcfg -u`も`0x8007007e`
    - ネットワーク部品を調べたら`vmware_bridge`が使う`vmnetbridge.dll`がSystem32になかった
- 原因: 古いVMware(Eドライブにあった)の削除が中途半端で、登録だけ残りDLLが消えていた
- 対処: `E:\VMwareWorkStationPro\vmnetBridge.dll`を`C:\Windows\System32\vmnetbridge.dll`にコピー > デフォルトに戻す
- 再発防止: ハイパーバイザーをソフトを消す場合はファイルから消すのではなくsystemから正式にアンインストールしてpath等も削除されるようにする。別々の失敗に同じエラーコードが出ていたら、共通の原因を疑う。

