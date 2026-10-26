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
