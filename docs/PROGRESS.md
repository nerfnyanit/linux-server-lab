# 進捗

タスクの詳細と完了条件は `SPEC.md` を参照。完了したら `[x]` にして日付を書く(例:`[x] 1-1 ... (10/23)`)。

## 現在地
- Step:1
- 次にやること:1-4
- 締切:**11/1(日)**
- 費用:**0円**(AWS無料プランのクレジット・無料リソース内、ドメインはDuckDNS)

## Step 1:手元VM(10/22〜10/25)
- [x] 1-1 VMware + AlmaLinux 9 + 固定IP (10/23)
- [x] 1-2 一般ユーザー + wheel(sudo) (10/23)
- [x] 1-3 SSH鍵のみ・root禁止 (10/23)
- [ ] 1-4 firewalld(SSH・HTTPのみ)
- [ ] 1-5 Nginxで静的ページ
- [ ] 1-6 自作アプリをsystemdで常駐
- [ ] 完了条件 1〜6 確認(SELinux Enforcing)

## Step 2:EC2で公開(10/26〜10/29)
- [ ] 事前準備:MFA・予算アラート・東京リージョン
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
- 10/23 `systemctl reroad` の打ち間違いで reload されず、パスワード認証が有効なままだった → troubleshooting.md に記録する(学び:`sshd -T` はファイルの値で、動いているプロセスの値ではない)
- 10/23 VMwareのホスト仮想アダプタが作れない障害(原因:旧VMwareの不完全な削除で `vmnetbridge.dll` が欠落)→ troubleshooting.md に記録する
