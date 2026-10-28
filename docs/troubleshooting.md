# 障害・エラー記録

起きたエラーや障害は、解決したら必ずここに追記する。新しいものを上に書く。

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

