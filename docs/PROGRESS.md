# 進捗

タスクの詳細と完了条件は `SPEC.md` を参照。完了したら `[x]` にして日付を書く(例:`[x] 1-1 ... (10/23)`)。

## 現在地
- Step:**全 Step 完了(10/30)**
- 次にやること:11/1 は予備日。任意で fail2ban / dnf-automatic、`poweredByHeader: false`。12/22 までに AWS リソースを削除
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
- [x] 2-1 EC2(AlmaLinux 9 AMI)作成・セキュリティグループ (10/27)
- [x] 2-2 Elastic IP (10/27)
- [x] 2-3 初期設定の適用 (10/27)
- [x] 2-4 Next.jsをsystemdで常駐 (10/27)
- [x] 2-5 Nginxリバースプロキシ + SELinux設定(httpd_can_network_connect) (10/28)
- [x] 2-6 DuckDNS (10/28)
- [x] 2-7 EPEL + certbot でHTTPS化・リダイレクト (10/28)
- [x] 完了条件 1〜5 確認(再起動後も nginx / portfolio / certbot-renew.timer が active、boolean on、スワップ有効) (10/28)

## Step 3:壊して直す・まとめ(10/30〜11/1)
- [x] 3-1 壊して直す訓練 a / b / c / d / e (10/30)(準備:VM の `/app/` → hello-app で 502 → boolean で解決 10/29。b 10/29、a・c・d・e 10/30。c はスクリプトの不具合で1回やり直し)
- [x] 3-2 切り分けコマンド (10/30)
- [x] 3-3 runbook `deploy.md` (10/30)
- [x] 3-4 README.md (10/30)
- [x] 完了条件:a〜e を自力で直せた / README で何をどう作ったかが伝わる (10/30)
- [ ] (任意)fail2ban / dnf-automatic

## 終わったあと
- [ ] 12/22より前にAWSリソースを削除(またはOracle Always Freeへ移行)

## メモ
- VM:VMnet8(NAT)のサブネット内で、DHCP の範囲外に固定 IP を設定、メモリ 2GB。再起動後も固定IPが残ることを確認済み
- VMのユーザー:普段使いの一般ユーザー(wheel)を作成して使う
- sshd の設定は `/etc/ssh/sshd_config.d/49-*.conf`(50-redhat.conf より先に読ませるため)。控えを `configs/` に置く。EC2 では cloud-init の `50-cloud-init.conf` にも注意
- Nginx のバージョン表示を隠す `server_tokens off;`(nginx.conf の http ブロック)は VM で設定済み(10/26)。EC2(2-5)でも同じ設定を入れる
- 自作アプリ:`/opt/hello-app/server.js`(127.0.0.1:3000、root 所有で 644)、ユニット `/etc/systemd/system/hello-app.service`(`User=hello`、システムユーザー uid 995、`Restart=always`)
- EC2:t3.micro、AlmaLinux 9.8 Marketplace 公式 AMI、EBS 20GiB gp3、Elastic IP、SG(22=自宅IPのみ / 80・443=全体)、キーペアは手元の `~/.ssh/` に保管。初期ユーザー `ec2-user`
- EC2 初期設定(10/27):`dnf update` → 再起動、普段使いの一般ユーザー(wheel、ed25519 の鍵)、sshd の設定(configs の控えを scp)、firewalld を追加インストールし起動前に `firewall-offline-cmd` で ssh/http/https を許可。cloud-init の `50-cloud-init.conf` は元から PasswordAuthentication no。メモリ 717Mi・スワップ無し → 2-4 の前にスワップを作る
- 12/22 までに削除するもの:EC2 → Elastic IP(解放)→ EBS → SG → キーペア
- EC2 の Next.js(10/27):スワップ 2G(`/swapfile`、`swapfile_t`、fstab 追記)、Node.js 24、システムユーザー `portfolio`(ホーム `/opt/portfolio`)、アプリ `/opt/portfolio/app`(`npm ci` → `npm run build`)、ユニット `/etc/systemd/system/portfolio.service`(`-H 127.0.0.1 -p 3000`、`NODE_ENV=production`、`NEXT_TELEMETRY_DISABLED=1`)
- ドメイン:`nerfnyanit-portfolio.duckdns.org` → `54.92.104.136`(DuckDNS の token は秘密情報。リポジトリに書かない)
- EC2 の Nginx(10/28):`/etc/nginx/conf.d/portfolio.conf`(最初は listen 80・`server_name _`、2-7 でドメインに変更。`proxy_pass http://127.0.0.1:3000`)。nginx.conf の既定の server ブロックはコメントアウト(バックアップ `nginx.conf.bak`)。502 → error.log `(13: Permission denied)` → `ausearch -m avc` で `httpd_t` → `ntop_port_t` の name_connect 拒否 → `setsebool -P httpd_can_network_connect 1` → troubleshooting.md に記録済み
- EC2 の HTTPS(10/28):`server_name` を `nerfnyanit-portfolio.duckdns.org` に変更 → EPEL + certbot(`certbot --nginx`)。証明書の期限 2027-01-26(90日)、秘密鍵 `/etc/letsencrypt/live/.../privkey.pem` は外に出さない。certbot が 443 ssl と 80→301 の server ブロックを追加(IP 直打ちは 404)。バックアップ `/etc/nginx/portfolio.conf.before-certbot`。80 番のブロックで `nginx/1.20.1` が見えていた → `server_tokens off` を nginx.conf の http ブロックに移した。`certbot-renew.timer` は自動で起動しないので `enable --now`。`renew --dry-run` 成功
- **TODO(開発環境で)**:`next.config.ts` に `poweredByHeader: false`(`X-Powered-By: Next.js` を消す)
- 10/27 脆弱性対応:`npm audit --omit=dev` で next 16.1.1 に critical(認証不要の RCE など)。開発環境で `npm audit fix`(--force なし)→ next 16.4.3、0 件。PR #6 をマージ、Vercel 本番は更新済み。EC2 にも取り込み済み(16.4.3、`npm audit --omit=dev` で 0 件、10/27)
- EC2 の設定の控え(10/28):`configs/systemd/portfolio.service`、`configs/nginx/portfolio.conf`、`configs/nginx/nginx.conf`(証明書・秘密鍵は含めない)
- ポートフォリオ:`nerfnyanit/frontend-portfolio`(public)。Next.js 16 / React 19。開いている PR は無し、main に最新がマージ済み(10/27 確認)
- AWS 予算 `monthly-10usd`(名前は変更不可):10/27 に **月 $20** に変更(EC2+EBS+IPv4 で月 約 $15 かかるため)。請求タイプ Credit を除外(実使用額で判定)、実際のコスト 80% / 100% でメール通知。EBS と Elastic IP は EC2 を停止しても課金される
- 10/26 ユニットファイルに `chmod a+rwx` してしまった件 → troubleshooting.md に記録済み
- 10/23 `systemctl reroad` の打ち間違いで reload されず、パスワード認証が有効なままだった → troubleshooting.md に記録する(学び:`sshd -T` はファイルの値で、動いているプロセスの値ではない)
- 10/23 VMwareのホスト仮想アダプタが作れない障害(原因:旧VMwareの不完全な削除で `vmnetbridge.dll` が欠落)→ troubleshooting.md に記録する
