# EC2 でゼロから公開する手順

## 0. 事前準備(AWS)
- AWS で root アカウントを作成し、MFA を設定(多要素認証で乗っ取り防止)
- Budgets で予算アラートを設定(月 $20、実際のコストの 80% / 100% でメール通知、Credit を除外)
- リージョンを東京にする(ap-northeast-1)

## 1. EC2 を作る(AMI・インスタンスタイプ・SG・キーペア)
- AMI は AlmaLinux 9(公式 AMI、無料)
- t3.micro(無料プランの対象になる小さいインスタンスタイプ)
- SG:22 は自分の IP のみ(`<自宅IP>/32`)、HTTP / HTTPS は全体に開放
- キーペアを作って、デスクトップ PC の秘密のフォルダに保存
- EBS のサイズと種類:20GiB gp3
- 初期ユーザーで接続:`ssh -i <鍵のパス> ec2-user@<Elastic IP>`

## 2. Elastic IP を割り当てる
- 割り当てて、インスタンスに関連付ける
- なぜ必要か:停止・起動で IP が変わるのを防ぐ
- 注意:使っていなくても課金される

## 3. 初期設定(dnf update、ユーザー作成、SSH 鍵、sshd、firewalld)

```bash
# 初期ユーザーでログイン
ssh -i ~/.ssh/portfolio-key.pem ec2-user@<Elastic IP>

# 更新して再起動
sudo dnf update -y
sudo reboot
```
- なぜ:最新バージョンのパッケージで動かし、既知の脆弱性を塞ぐため。カーネルも更新されるので再起動する

```bash
# 普段使いのユーザーを作り、wheel に入れる
sudo useradd uchida
sudo passwd uchida
sudo usermod -aG wheel uchida
```
- なぜ root や ec2-user ではなく一般ユーザー + sudo か:root ユーザーだと複数人で管理したときに、誰が操作したかがログで特定できない。一般ユーザー + sudo であれば、どのユーザーが使ったかを `/var/log/secure` で特定できる。うっかりミスでシステム全体を壊しにくくもなる

```bash
# 公開鍵を置く(手元の id_ed25519.pub の中身を authorized_keys に)
sudo mkdir -p /home/uchida/.ssh
sudo vi /home/uchida/.ssh/authorized_keys
sudo chmod 700 /home/uchida/.ssh
sudo chmod 600 /home/uchida/.ssh/authorized_keys
sudo chown -R uchida:uchida /home/uchida/.ssh
```
- なぜ 700 / 600 にするか:持ち主(uchida)だけが読み書きできるようにするため。他人が `authorized_keys` に自分の鍵を書き足せたら、uchida としてログインできてしまう。sshd は権限がゆるいと、その鍵を使わせない

```bash
# sshd の設定(configs/sshd_config.d/49-uchida.conf を置く)
scp -i ~/.ssh/portfolio-key.pem configs/sshd_config.d/49-uchida.conf ec2-user@<Elastic IP>:/tmp/
sudo mv /tmp/49-uchida.conf /etc/ssh/sshd_config.d/
sudo restorecon -v /etc/ssh/sshd_config.d/49-uchida.conf
sudo sshd -t
sudo systemctl reload sshd
```
- 内容:鍵認証のみ・root ログイン禁止
- なぜファイル名を 49- にするか:先頭の数字が小さいほど先に読み込まれる。sshd は同じ項目が何度出てきても最初に読んだ値を使うので、`50-redhat.conf` や `50-cloud-init.conf` より先に読ませて、自分の設定を優先させる
- 注意:**今のセッションは閉じずに**、別のターミナルで `ssh uchida@<Elastic IP>` できることを確認してから閉じる
- なぜ `restorecon` が必要か:`mv` で移したファイルには、`/tmp` にいたときの SELinux のラベルが付いたままになる。そのままだと sshd が読めないので、置き場所に合ったラベルに付け直す

```bash
# firewalld(AMI に入っていないので入れる)
sudo dnf install -y firewalld
sudo firewall-offline-cmd --add-service=ssh --add-service=http --add-service=https
sudo systemctl enable --now firewalld
sudo firewall-cmd --list-all
```
- なぜ起動する前に `firewall-offline-cmd` で ssh を許可しておくか:firewalld が停止している状態でも、設定を直接永続的に書き換えられる。ssh を許可しないまま起動すると、今つないでいる SSH が切られて、手元に無い EC2 に二度と入れなくなるおそれがある

## 4. スワップを作る

```bash
sudo dd if=/dev/zero of=/swapfile bs=1M count=2048
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap defaults 0 0' | sudo tee -a /etc/fstab
free -h
```
- なぜスワップが必要か(t3.micro のメモリは約 1GB):`npm run build` などでメモリが足りなくなると、プロセスが強制終了される(OOM)。足りない分をディスクで補う
- なぜ fstab に書くか:`swapon` だけだと、再起動するとスワップが外れる。fstab に書くと、起動時に自動で有効になる

## 5. Node.js と Next.js(アプリのビルドと systemd 化)

```bash
# Node.js 24 と git
dnf module list nodejs              # 使えるバージョンを確認
sudo dnf module install nodejs:24
sudo dnf install -y git
node -v

# アプリ専用のシステムユーザー(ログイン不可)
sudo useradd --system --create-home --home-dir /opt/portfolio --shell /sbin/nologin portfolio

# 取得してビルド
sudo -u portfolio git clone https://github.com/nerfnyanit/frontend-portfolio.git /opt/portfolio/app
cd /opt/portfolio/app
sudo -u portfolio npm ci
sudo -u portfolio npm run build

# systemd に登録(configs/systemd/portfolio.service を置く)
sudo vi /etc/systemd/system/portfolio.service
sudo systemctl daemon-reload
sudo systemctl enable --now portfolio
curl -I http://127.0.0.1:3000
```
- なぜアプリ専用のユーザーで動かすか:アプリが乗っ取られても、触れるのは portfolio ユーザーの範囲だけにするため(最小権限)。ログインもできない
- なぜ `127.0.0.1`(`-H 127.0.0.1`)で待ち受けるか:サーバーの中からしか届かないようにして、外の入口を Nginx だけにするため
- `Restart=always` と `enable` は何のためか:`Restart=always` はアプリが落ちても自動で起動し直す。`enable` はサーバーを再起動したときに自動で起動する

## 6. Nginx(リバースプロキシ)と SELinux

```bash
sudo dnf install -y nginx

# 既定の server ブロックをコメントアウトし、http ブロックに server_tokens off;(configs/nginx/nginx.conf)
sudo cp -p /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bak
sudo vi /etc/nginx/nginx.conf

# リバースプロキシの設定(configs/nginx/portfolio.conf の 80 番の部分。443 は 8 で certbot が足す)
sudo vi /etc/nginx/conf.d/portfolio.conf

sudo nginx -t
sudo systemctl enable --now nginx

# SELinux:nginx から 3000 番への接続を許可
sudo setsebool -P httpd_can_network_connect 1
getsebool httpd_can_network_connect
curl -I http://127.0.0.1/
```
- なぜ `nginx -t` を先にやるか:書き間違いがあるまま起動・再起動すると、nginx が起動できずにサイトが止まる。`-t` ならファイル名と行番号まで教えてくれる
- `server_tokens off` は何のためか:エラーページやヘッダーに nginx のバージョンを出さないため。バージョンがわかると、そのバージョンの脆弱性を狙われやすい
- boolean を設定しないと何が起きるか:SELinux が nginx(`httpd_t`)から 3000 番(`ntop_port_t`)への接続を拒否し、502 Bad Gateway になる
- なぜ SELinux を無効にしないか:無効にするとアクセス制御そのものがなくなる。boolean で必要なことだけを許可すれば、ほかの守りは残せる

## 7. DuckDNS(ドメイン)
- DuckDNS にログインし、サブドメイン `nerfnyanit-portfolio` を作る
- IP に `<Elastic IP>` を設定する
- `nslookup nerfnyanit-portfolio.duckdns.org` で Elastic IP が返ることを確認
- 注意:DuckDNS の token は秘密情報。リポジトリに書かない
- なぜ DuckDNS か:無料でサブドメインが使えるため(費用 0 円)。Let's Encrypt の証明書はドメインに対して発行されるので、HTTPS にするにはドメインが必要

## 8. HTTPS(EPEL + certbot、自動更新)

```bash
# server_name をドメインに変えてから
sudo vi /etc/nginx/conf.d/portfolio.conf
sudo nginx -t && sudo systemctl reload nginx

sudo dnf install -y epel-release
sudo dnf install -y certbot python3-certbot-nginx
sudo certbot --nginx -d nerfnyanit-portfolio.duckdns.org

# 自動更新(タイマーは自動では起動しない)
sudo systemctl enable --now certbot-renew.timer
systemctl list-timers | grep certbot
sudo certbot renew --dry-run
```
- なぜ EPEL が必要か:certbot は AlmaLinux の標準リポジトリに無く、EPEL(追加パッケージのリポジトリ)にあるため
- `certbot --nginx` は nginx の設定に何を足すか:443 番で HTTPS を受ける server ブロック(証明書と秘密鍵の場所)と、80 番に来たアクセスを 301 で HTTPS に転送する設定
- なぜ自動更新が必要か(証明書の期限は何日か):Let's Encrypt の証明書は 90 日で切れる。切れるとブラウザに警告が出て、サイトが見られなくなる
- 注意:秘密鍵 `privkey.pem` はサーバーの外に出さない

## 9. 完了確認

```bash
# 手元の PC から
curl -I https://nerfnyanit-portfolio.duckdns.org   # 200
curl -I http://nerfnyanit-portfolio.duckdns.org    # 301 → https
Test-NetConnection <Elastic IP> -Port 3000         # (PowerShell)失敗すれば OK

# サーバーを再起動してから
sudo reboot
systemctl is-active nginx portfolio certbot-renew.timer   # すべて active
getsebool httpd_can_network_connect                       # on
free -h                                                   # スワップ 2G
sudo ss -tlnp                                             # 3000 は 127.0.0.1 だけ
```
- なぜ 3000 番に外から届かないことを確認するか:SG・firewalld・`127.0.0.1` の多層防御が、設定どおり効いているかを外から確かめるため
- なぜ再起動してから確認するか:`enable` や `-P`、fstab の書き忘れは、再起動して初めて気づく。再起動しても全部戻ることを確かめる

## 10. 片付け(削除する順番)
1. EC2 インスタンスを終了(terminate)
2. Elastic IP を解放(release)
3. EBS ボリュームが残っていれば削除
4. セキュリティグループを削除
5. キーペアを削除
- 最後に請求ダッシュボードで、課金が止まったことを確認する
- なぜ Elastic IP の解放を忘れてはいけないか:EC2 を消しても Elastic IP は残り、使っていなくても課金され続けるため
