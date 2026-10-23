# Linux Server Lab 仕様書(10日間・軽量版)

## 1. ゴール
**11/1 までに、自分の Next.js ポートフォリオを、自分で立てた Linux サーバー(AWS EC2)で HTTPS 公開する。**
あわせて、最低限の安全設定と「繋がらない」ときの切り分けができるようになる。

## 2. 前提
- 期間:2026/10/22(木)〜11/1(日)
- **費用0円**:AWS は無料プランのクレジット内(有料プランにはアップグレードしない)、ドメインは無料の DuckDNS
- AWS リージョンは東京(ap-northeast-1)
- 仮想化ソフトは VMware Workstation Pro(個人利用は無料)
- OS は **AlmaLinux 9**(RHEL系)を手元VM・EC2の両方で使う。EC2 で AlmaLinux の AMI が選べない場合は Amazon Linux 2023 で代用する
- **SELinux は Enforcing のまま進める**(無効化しない)。現場の RHEL 系サーバーは有効が普通のため

## 3. スケジュール
| Step | 期間 | 内容 | ゴール |
|---|---|---|---|
| 1 | 10/22(木)〜10/25(日) | 手元VM(VMware + AlmaLinux)で基本を立てる | 鍵認証のみでSSH、Nginxでページ表示、自作アプリをsystemdで常駐 |
| 2 | 10/26(月)〜10/29(木) | EC2でポートフォリオを公開 | `https://<名前>.duckdns.org` でポートフォリオが見える |
| 3 | 10/30(金)〜11/1(日) | 壊して直す・手順書・README | 障害を自力で直せる、READMEで説明できる |

---

## Step 1:手元VMで基本を立てる(10/22〜10/25)

### タスク
- [1-1] VMware に AlmaLinux 9(最小インストール)を入れる。ネットワークは NAT(VMnet8)にし、`nmcli` で固定IPを振る。ホストから ping と SSH が届くことを確認する
- [1-2] 一般ユーザーを作り、`wheel` グループに入れて sudo 権限を付ける
- [1-3] ホストで SSH 鍵(ed25519)を作って VM に登録し、sshd を「鍵認証のみ・root ログイン禁止」にする
- [1-4] firewalld で SSH と HTTP(80)だけを許可する(`--permanent` を付けて再起動後も残す)
- [1-5] Nginx を `dnf` で入れて、自作の静的 HTML ページを表示する。SELinux は Enforcing のまま
- [1-6] Node.js の小さい HTTP サーバーを作り、自分で書いた systemd ユニットで常駐させる(自動起動・落ちたら再起動)

### 完了条件
| # | 条件 | 確認コマンド例 |
|---|---|---|
| 1 | 鍵で SSH できる | `ssh user@<VM IP>` |
| 2 | パスワードと root では SSH できない | `ssh -o PubkeyAuthentication=no user@<VM IP>` / `ssh root@<VM IP>` が拒否される |
| 3 | 許可したポートだけ開いている | `sudo firewall-cmd --list-all` |
| 4 | ホストから静的ページが見える | `curl -I http://<VM IP>` が 200 |
| 5 | 再起動・kill しても自作アプリが自動で上がる | `sudo reboot` / `sudo kill <PID>` 後に `systemctl status <service>` |
| 6 | SELinux が有効なまま動いている | `getenforce` が Enforcing |

---

## Step 2:EC2 でポートフォリオを公開(10/26〜10/29)

### 事前準備(最初に必ず)
- ルートユーザーに MFA を設定する
- 予算アラートを設定する(例:$10 で通知)
- リージョンを東京にする

### タスク
- [2-1] EC2(AlmaLinux 9 の公式AMI、無料プランで選べる小さいインスタンス)を作る。AMI に追加料金がかからないことを確認する。初期ユーザー名は AMI によって違うので確認する(AlmaLinux は `ec2-user`)。セキュリティグループは **SSH は自分のIPのみ**、HTTP/HTTPS は全体に開放
- [2-2] Elastic IP を付けて IP を固定する
- [2-3] Step 1 の初期設定(ユーザー、鍵認証のみ、root 禁止、firewalld)を EC2 にも適用する。EC2 ではセキュリティグループと firewalld の二重になることを意識する
- [2-4] Node.js を入れ(`dnf module` で新しめのバージョンを選ぶ)、Next.js ポートフォリオを `git clone` → ビルドし、systemd で常駐させる(ポート 3000 は外に公開しない)
- [2-5] Nginx をリバースプロキシにして、80 番 → localhost:3000 に流す。**SELinux が Nginx から 3000 番への接続を止めるので、`httpd_can_network_connect` を有効にする**(SELinux を無効にして逃げない)
- [2-6] DuckDNS でサブドメインを取り、Elastic IP に向ける
- [2-7] EPEL リポジトリを有効にして certbot を入れ、Let's Encrypt で HTTPS 化し、HTTP → HTTPS にリダイレクトする

### 完了条件
| # | 条件 | 確認方法 |
|---|---|---|
| 1 | `https://<名前>.duckdns.org` でポートフォリオが見える | ブラウザ / `curl -I https://...` |
| 2 | HTTP が HTTPS にリダイレクトされる | `curl -I http://...` が 301 |
| 3 | 3000 番が外から見えない | ホストから `nc -zv <IP> 3000` が失敗 |
| 4 | 証明書の自動更新が通る | `sudo certbot renew --dry-run` |
| 5 | EC2 を再起動してもサイトが自動で復活する | `sudo reboot` 後にブラウザで確認 |

---

## Step 3:壊して直す・まとめ(10/30〜11/1)

### タスク
- [3-1] **壊して直す訓練(手元VMのみ)**:Claude が次の障害を1つずつ仕込み、本人が切り分けて直す。直したら `docs/troubleshooting.md` に記録する
  - a. Nginx が止まっている
  - b. firewalld で 80 番が閉じている
  - c. Nginx の設定ファイルに書き間違いがある
  - d. 自作アプリのサービスが落ちて上がらない
  - e. SELinux に止められて 502 Bad Gateway になる(`ausearch` / `/var/log/audit/audit.log` で原因を見つける)
- [3-2] 切り分けで使うコマンドに慣れる:`systemctl status`、`journalctl -u`、`ss -tlnp`、`curl -v`、`sudo nginx -t`、`sudo firewall-cmd --list-all`、`getenforce`、`sudo ausearch -m avc -ts recent`
- [3-3] `docs/runbooks/deploy.md` に、ゼロから EC2 で公開するまでの手順を自分の言葉で書く
- [3-4] `README.md` を書く:構成図(ブラウザ → Nginx → Next.js)、やったこと、**なぜそうしたか**(なぜ鍵認証のみか、なぜ 3000 番を外に出さないか、など)

### 完了条件
- a〜e を、答えを見ずに自力で直せた
- README を読めば、何をどう作ったかが他人に伝わる

### 余裕があれば(任意)
- fail2ban で SSH への総当たりを遮断する
- dnf-automatic で自動セキュリティアップデートを設定する

---

## 4. 終わったあと
- **12/22(無料プラン終了)より前に、EC2・Elastic IP などの AWS リソースを削除する**。公開を続けたい場合は、無料の Oracle Cloud Always Free に引っ越す(手順書どおりに作り直せるかの良い練習になる)
- 続けたくなったら発展編へ:PostgreSQL 追加、バックアップとリストア、監視、Ubuntu(Debian系)との違い、Docker、Ansible
