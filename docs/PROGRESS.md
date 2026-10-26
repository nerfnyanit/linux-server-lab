# 進捗

タスクの詳細と完了条件は `SPEC.md` を参照。完了したら `[x]` にして日付を書く(例:`[x] 1-1 ... (10/23)`)。

## 現在地
- Step:2
- 次にやること:2-1
- 締切:**11/1(日)**
- 費用:**0円**(AWS無料プランのクレジット・無料リソース内、ドメインはDuckDNS)

## Step 1:手元VM(10/22〜10/25)
- [x] 1-1 VMware + AlmaLinux 9 + 固定IP (10/23)
- [x] 1-2 一般ユーザー + wheel(sudo) (10/23)
- [x] 1-3 SSH鍵のみ・root禁止 (10/23)
- [x] 1-4 firewalld(SSH・HTTPのみ) (10/26)
- [x] 1-5 Nginxで静的ページ (10/26)
- [x] 1-6 自作アプリをsystemdで常駐 (10/26)
- [x] 完了条件 1〜6 確認(SELinux Enforcing) (10/26)

## Step 2:EC2で公開(10/26〜10/29)
- [x] 事前準備:MFA・予算アラート・東京リージョン (10/26)
- [ ] 2-1 EC2(AlmaLinux 9 AMI)作成・セキュリティグループ
- [ ] 2-2 Elastic IP
- [ ] 2-3 初期設定の適用
- [ ] 2-4 Next.jsをsystemdで常駐
- [ ] 2-5 Nginxリバースプロキシ + SELinux設定(httpd_can_network_connect)
- [ ] 2-6 DuckDNS
- [ ] 2-7 EPEL + certbot でHTTPS化・リダイレクト
- [ ] 完了条件 1〜5 確認

## Step 3:壊して直す・まとめ(10/30〜11/1)
- [ ] 3-1 壊して直す訓練 a / b / c / d / e
- [ ] 3-2 切り分けコマンド
- [ ] 3-3 runbook `deploy.md`
- [ ] 3-4 README.md
- [ ] (任意)fail2ban / dnf-automatic

## 終わったあと
- [ ] 12/22より前にAWSリソースを削除(またはOracle Always Freeへ移行)

## メモ
- VM:`192.168.99.100/24`(VMnet8 NAT、GW/DNS `192.168.99.2`、DHCP範囲 `.128〜.254`)、メモリ 2GB。再起動後も固定IPが残ることを確認済み
- VMのファイルは E ドライブ(20GB)。設定は `E:\VM_AlmaLinux`、ディスクは `E:\LinuxVM`(**消さない**)。空きが 3GB を切ったら VM を D に移す
- VMのユーザー:`uchida`(uid 1001、wheel)を普段使いにする。`admin`(インストーラーで作成、wheel)も残っている
- sshd の設定は `/etc/ssh/sshd_config.d/49-uchida.conf`(50-redhat.conf より先に読ませるため)。控えを `configs/` に置く。EC2 では cloud-init の `50-cloud-init.conf` にも注意
- Nginx のバージョン表示を隠す `server_tokens off;`(nginx.conf の http ブロック)は VM で設定済み(10/26)。EC2(2-5)でも同じ設定を入れる
- 自作アプリ:`/opt/hello-app/server.js`(127.0.0.1:3000、root 所有で 644)、ユニット `/etc/systemd/system/hello-app.service`(`User=hello`、システムユーザー uid 995、`Restart=always`)
- AWS 予算 `monthly-10usd`:月 $10、請求タイプ Credit を除外(実使用額で判定)、実際のコスト 80% / 100% でメール通知。EBS と Elastic IP は EC2 を停止しても課金される
- 10/26 ユニットファイルに `chmod a+rwx` してしまった件 → troubleshooting.md に記録済み
- 10/23 `systemctl reroad` の打ち間違いで reload されず、パスワード認証が有効なままだった → troubleshooting.md に記録する(学び:`sshd -T` はファイルの値で、動いているプロセスの値ではない)
- 10/23 VMwareのホスト仮想アダプタが作れない障害(原因:旧VMwareの不完全な削除で `vmnetbridge.dll` が欠落)→ troubleshooting.md に記録する
