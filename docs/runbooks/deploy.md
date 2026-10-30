# EC2 でゼロから公開する手順

- 【手元】は手元の PC(PowerShell)、【EC2】は EC2 に SSH でつないだ状態で実行する
- `<Elastic IP>`、`<自宅IP>`、`<user>`(普段使いのユーザー名)は、自分の値に置き換える
- 設定ファイルの控えは、このリポジトリの `configs/` にある。最初に手元に取得しておく

```bash
# 【手元】このリポジトリを取得して、作業ディレクトリにする
git clone https://github.com/nerfnyanit/linux-server-lab.git
cd linux-server-lab
```

## 0. 事前準備(AWS)
- AWS で root アカウントを作成し、MFA を設定(多要素認証で乗っ取り防止)
- Budgets で予算アラートを設定(月 $20、実際のコストの 80% / 100% でメール通知、Credit を除外)。**通知だけで、自動で止まるわけではない**
- リージョンを東京にする(ap-northeast-1)

## 1. EC2 を作る(AMI・インスタンスタイプ・SG・キーペア)
- AMI は AlmaLinux 9(公式 AMI。AMI 自体の料金は無し)
- t3.micro(無料プランの対象になる小さいインスタンスタイプ。ただし利用料は発生し、クレジットから引かれる)
- SG:22 は自分の IP のみ(`<自宅IP>/32`)、HTTP / HTTPS は全体に開放
- キーペアを作って、手元 PC の `~/.ssh/` に保存
- EBS のサイズと種類:20GiB gp3

## 2. Elastic IP を割り当てる
- 割り当てて、インスタンスに関連付ける
- なぜ必要か:停止・起動で IP が変わるのを防ぐ
- 注意:使っていなくても課金される

## 3. 初期設定(dnf update、ユーザー作成、SSH 鍵、sshd、firewalld)

```bash
# 【手元】初期ユーザーでログイン
ssh -i ~/.ssh/<キーペア>.pem ec2-user@<Elastic IP>

# 【EC2】更新して再起動
sudo dnf update -y
sudo reboot
# 【手元】数十秒待ってから、もう一度 ssh でつなぎ直す
```
- なぜ:最新バージョンのパッケージで動かし、既知の脆弱性を塞ぐため。カーネルも更新されるので再起動する

```bash
# 【EC2】普段使いのユーザーを作り、wheel に入れる
sudo useradd <user>
sudo passwd <user>
sudo usermod -aG wheel <user>
```
- なぜ root や ec2-user ではなく一般ユーザー + sudo か:root ユーザーだと複数人で管理したときに、誰が操作したかがログで特定できない。一般ユーザー + sudo であれば、どのユーザーが使ったかを `/var/log/secure` で特定できる。うっかりミスでシステム全体を壊しにくくもなる

```bash
# 【EC2】公開鍵を置く(手元の ~/.ssh/id_ed25519.pub の中身を authorized_keys に貼る)
sudo mkdir -p /home/<user>/.ssh
sudo vi /home/<user>/.ssh/authorized_keys
sudo chmod 700 /home/<user>/.ssh
sudo chmod 600 /home/<user>/.ssh/authorized_keys
sudo chown -R <user>:<user> /home/<user>/.ssh

# 【手元】別のウィンドウで、新しいユーザーで鍵ログインできることを確認
ssh <user>@<Elastic IP>
```
- なぜ 700 / 600 にするか:持ち主(`<user>`)だけが読み書きできるようにするため。他人が `authorized_keys` に自分の鍵を書き足せたら、そのユーザーとしてログインできてしまう。sshd は権限がゆるいと、その鍵を使わせない

```bash
# 【手元】sshd の設定ファイルを送る(configs/sshd_config.d/49-uchida.conf。ファイル名は 49- で始まれば何でもよい)
scp -i ~/.ssh/<キーペア>.pem configs/sshd_config.d/49-uchida.conf ec2-user@<Elastic IP>:/tmp/

# 【EC2】置いて、ラベルを直し、文法を確認してから reload
sudo mv /tmp/49-uchida.conf /etc/ssh/sshd_config.d/
sudo chown root:root /etc/ssh/sshd_config.d/49-uchida.conf
sudo chmod 600 /etc/ssh/sshd_config.d/49-uchida.conf
sudo restorecon -v /etc/ssh/sshd_config.d/49-uchida.conf
sudo sshd -t
sudo systemctl reload sshd
sudo systemctl status sshd   # Reloaded の記録を確認

# 【EC2】実際に効いている値を確認する(期待値:passwordauthentication no / kbdinteractiveauthentication no / permitrootlogin no / pubkeyauthentication yes)
sudo sshd -T | grep -Ei '^(passwordauthentication|kbdinteractiveauthentication|permitrootlogin|pubkeyauthentication) '
```
- 内容:パスワード認証の無効化・root ログイン禁止。keyboard-interactive 認証は AlmaLinux 既定の `50-redhat.conf` で無効になっているが、上の `sshd -T` で必ず確認する
- 注意:**今のセッションは閉じずに**、別のウィンドウで次の確認をしてから閉じる

```bash
# 【手元】鍵でログインできる
ssh <user>@<Elastic IP>
# 【手元】鍵を使わない認証は拒否される(Permission denied (publickey...) になれば OK)
ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password,keyboard-interactive <user>@<Elastic IP>
# 【手元】root では入れない
ssh root@<Elastic IP>
```
- なぜファイル名を 49- にするか:先頭の数字が小さいほど先に読み込まれる。sshd は同じ項目が何度出てきても最初に読んだ値を使うので、`50-redhat.conf` や `50-cloud-init.conf` より先に読ませて、自分の設定を優先させる
- なぜ `restorecon` が必要か:`mv` で移したファイルには、`/tmp` にいたときの SELinux のラベルが付いたままになる。そのままだと sshd が読めないので、置き場所に合ったラベルに付け直す

```bash
# 【EC2】firewalld(AMI に入っていないので入れる)
sudo dnf install -y firewalld
sudo firewall-offline-cmd --add-service=ssh --add-service=http --add-service=https
sudo systemctl enable --now firewalld
sudo firewall-cmd --list-all
```
- なぜ起動する前に `firewall-offline-cmd` で ssh を許可しておくか:firewalld が停止している状態でも、設定を直接永続的に書き換えられる。ssh を許可しないまま起動すると、今つないでいる SSH が切られて、手元に無い EC2 に二度と入れなくなるおそれがある

## 4. スワップを作る

```bash
# 【EC2】
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
# 【EC2】Node.js 24 と git
dnf module list nodejs              # 使えるバージョンを確認
sudo dnf module install nodejs:24
sudo dnf install -y git
node -v

# 【EC2】アプリ専用のシステムユーザー(ログイン不可)
sudo useradd --system --create-home --home-dir /opt/portfolio --shell /sbin/nologin portfolio

# 【EC2】取得してビルド
sudo -u portfolio git clone https://github.com/nerfnyanit/frontend-portfolio.git /opt/portfolio/app
cd /opt/portfolio/app
sudo -u portfolio npm ci
sudo -u portfolio npm run build

# 【EC2】systemd に登録(中身は configs/systemd/portfolio.service と同じにする)
sudo vi /etc/systemd/system/portfolio.service
sudo systemctl daemon-reload
sudo systemctl enable --now portfolio
curl -I http://127.0.0.1:3000      # 200 が返れば OK
```
- なぜアプリ専用のユーザーで動かすか:アプリが乗っ取られても、触れるのは portfolio ユーザーの範囲だけにするため(最小権限)。ログインもできない
- なぜ `127.0.0.1`(`-H 127.0.0.1`)で待ち受けるか:サーバーの中からしか届かないようにして、外の入口を Nginx だけにするため
- `Restart=always` と `enable` は何のためか:`Restart=always` はアプリが落ちても自動で起動し直す。`enable` はサーバーを再起動したときに自動で起動する

## 6. Nginx(リバースプロキシ)と SELinux

```bash
# 【EC2】
sudo dnf install -y nginx

# 既定の server ブロックをコメントアウトし、http ブロックに server_tokens off;(configs/nginx/nginx.conf と同じにする)
sudo cp -p /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bak
sudo vi /etc/nginx/nginx.conf

# 証明書を取る前の、80 番だけの設定(中身は configs/nginx/portfolio.http.conf と同じにする)
# 最初は server_name を _ にして、IP で動作確認する
sudo vi /etc/nginx/conf.d/portfolio.conf

sudo nginx -t
sudo systemctl enable --now nginx

# SELinux:nginx から 3000 番への接続を許可(これが無いと 502 になる)
sudo setsebool -P httpd_can_network_connect 1
getsebool httpd_can_network_connect
curl -I http://127.0.0.1/          # 200 が返れば OK

# 【手元】外から HTTP で表示できることを確認
curl -I http://<Elastic IP>/
```
- なぜ `nginx -t` を先にやるか:書き間違いがあるまま起動・再起動すると、nginx が起動できずにサイトが止まる。`-t` ならファイル名と行番号まで教えてくれる
- `server_tokens off` は何のためか:エラーページやヘッダーに nginx のバージョンを出さないため。バージョンがわかると、そのバージョンの脆弱性を狙われやすい
- boolean を設定しないと何が起きるか:SELinux が nginx(`httpd_t`)から 3000 番(`ntop_port_t`)への接続を拒否し、502 Bad Gateway になる
- なぜ SELinux を無効にしないか:無効にすると SELinux によるアクセス制御がなくなる。boolean なら、必要になった種類の許可だけを足して、ほかのポリシーは残せる。ただし `httpd_can_network_connect` は、Web サーバーからどのポートへの接続も許可する点に注意

## 7. DuckDNS(ドメイン)
- 【手元】DuckDNS にログインし、サブドメイン `nerfnyanit-portfolio` を作る
- IP に `<Elastic IP>` を設定する
- 【手元】`nslookup nerfnyanit-portfolio.duckdns.org` で Elastic IP が返ることを確認
- 注意:DuckDNS の token は秘密情報。リポジトリに書かない
- なぜ DuckDNS か:無料でサブドメインが使えるため。Let's Encrypt の証明書はドメインに対して発行されるので、HTTPS にするにはドメインが必要

## 8. HTTPS(EPEL + certbot、自動更新)

順番は「**HTTP でドメインの表示を確認 → 証明書を取る → HTTPS で表示を確認**」。

```bash
# 【EC2】server_name を _ からドメインに変える
sudo vi /etc/nginx/conf.d/portfolio.conf
sudo nginx -t && sudo systemctl reload nginx

# 【手元】ドメインで HTTP 表示できることを確認(certbot の HTTP-01 認証は 80 番で行われる)
curl -I http://nerfnyanit-portfolio.duckdns.org

# 【EC2】証明書を取る(portfolio.conf に 443 と 80→301 の設定が追加される。結果は configs/nginx/portfolio.conf)
sudo dnf install -y epel-release
sudo dnf install -y certbot python3-certbot-nginx
sudo certbot --nginx -d nerfnyanit-portfolio.duckdns.org

# 【EC2】自動更新(タイマーは自動では起動しない)
sudo systemctl enable --now certbot-renew.timer
systemctl list-timers | grep certbot
sudo certbot renew --dry-run

# 【手元】HTTPS で表示でき、HTTP は HTTPS に転送されることを確認
curl -I https://nerfnyanit-portfolio.duckdns.org
curl -I http://nerfnyanit-portfolio.duckdns.org
```
- なぜ EPEL が必要か:certbot は AlmaLinux の標準リポジトリに無く、EPEL(追加パッケージのリポジトリ)にあるため
- `certbot --nginx` は nginx の設定に何を足すか:443 番で HTTPS を受ける server ブロック(証明書と秘密鍵の場所)と、80 番に来たアクセスを 301 で HTTPS に転送する設定
- なぜ自動更新が必要か(証明書の期限は何日か):Let's Encrypt の証明書は 90 日で切れる。切れるとブラウザに警告が出て、サイトが見られなくなる
- 注意:秘密鍵 `privkey.pem` はサーバーの外に出さない

## 9. 完了確認

```bash
# 【手元】
curl -I https://nerfnyanit-portfolio.duckdns.org   # 200
curl -I http://nerfnyanit-portfolio.duckdns.org    # 301 → https
Test-NetConnection <Elastic IP> -Port 3000         # (PowerShell)失敗すれば OK

# 【EC2】サーバーを再起動
sudo reboot

# 【手元】数十秒待ってから、もう一度 ssh でつなぎ直し、【EC2】で確認
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
