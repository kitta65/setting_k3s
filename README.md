# clone
```
git clone https://github.com/dr666m1/setting_k3s.git $HOME/.setting_k3s
```

# VMの作成
同じ設定のサーバーを2台用意する。
デフォルトからの変更点は以下。

- OSをUbuntu18.04に変更
- パブリックIPアドレスの割り当ては要検討（もしかしたらagent側にはいらないかも）
- SSHは公開キー・ファイルの選択から、ローカルの`id_rsa.pub`をアップロードする

# IPアドレス
## 予約済みIPアドレスの作成
- `コア・インフラストラクチャ` > `ネットワーキング` > `IP Management` > `パブリックIPアドレスの予約`
- `IPアドレス名`は任意、`コンパートメントに作成`はいったんルート、`IPアドレス・ソース`はいったんOracleにしておく

## 予約済みIPアドレスの割り当て
2台用意したサーバーの内、片方に割り当てる。

- VMの管理画面で`リソース` > `アタッチされたVNIC` から、表示中のVNICを選択
- VNICの管理画面で`リソース` > `IPアドレス`から、表示中のIPアドレスを編集 > 既存の予約済みIPアドレスの選択で、先ほど作成したIPアドレスを選択


# ssh接続
ユーザー名が`ubuntu`（Ubuntu以外は`opc`）なので`~/.ssh/config`を以下のようにするれば、
`ssh oracle` `ssh oracle-worker`だけでssh接続できる。
なお前者は予約済みIPアドレス（`xxx.xxx.xx.xx`）を割り当てたもので、
後者はプライベートIPアドレス（`yyy.yyy.yy.yy`）だけでどうにかなるか試したい。

```
Host oracle
    HostName xxx.xxx.xx.xx
    User ubuntu

Host oracle-agent
    HostName yyy.yyy.yy.yy
    User ubuntu
    ProxyCommand ssh -W %h:%p oracle
```

# NSG（Network Security Group）
iptablesもあるから、そこまで頑張らなくていいかも。

- `ネットワーキング` > `仮想クラウド・ネットワーク` > 表示中のVCNを選択 > `リソース` > `ネットワーク・セキュリティ・グループ`

# iptablesの設定
- netfilter-persistent経由でiptablesの設定が行われているらしい。
- 以下のコマンドで、設定を上書きする。

```
cat $HOME/.setting_k3s/rules.v4 | sudo tee /etc/iptables/rules.v4
sudo /etc/init.d/netfilter-persistent reload
```

- デフォルトの設定に以下を追記している（ドキュメントの該当部分は[ここ](https://rancher.com/docs/k3s/latest/en/installation/installation-requirements/#networking)）
- kubernetesが6443, 8472, 10250
- 8501は当初はstreamlit

```
-A INPUT -p tcp --dport  6443 -j ACCEPT
-A INPUT -p udp --dport  8472 -j ACCEPT
-A INPUT -p tcp --dport 10250 -j ACCEPT
-A INPUT -p tcp --dport  8501 -j ACCEPT
```


# k3sの設定
[日本語ドキュメント](https://rancher.co.jp/pdfs/K3s-eBook4Styles0507.pdf)が参考になる。
`kubeclt`で[`sudo`を使わなくてすむインストール方法](https://rancher.com/docs/k3s/latest/en/installation/install-options/how-to-flags/#example-a-k3s-kubeconfig-mode)もあるようだが、
[ここ](https://github.com/rancher/k3s/issues/389)のやりとり見ると推奨されていないっぽい...？
masterノードで何も実行したくない場合は、[古いドキュメント](https://www.rancher.co.jp/docs/k3s/latest/en/installation/)によると`curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable-agent" sh -`
を使えるが、[isuue](https://github.com/rancher/k3s/issues/978)によると推奨されておらず、taintを使うべき。

```sh
# oracle
curl -sfL https://get.k3s.io | sh -

# oracle-worker
curl -sfL https://get.k3s.io | K3S_URL=https://xxx.xxx.xx.xx:6443 K3S_TOKEN=mynodetoken sh -
```
mynodetokenはサーバー側で`sudo cat /var/lib/rancher/k3s/server/node-token`を実行して確認。


# helm
- [ここ](https://helm.sh/docs/intro/quickstart/)見てやってみる
- `helm repo add stable ...`はどうせやらないから不要かも

```
curl https://baltocdn.com/helm/signing.asc | sudo apt-key add -
sudo apt-get install apt-transport-https --yes
echo "deb https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm

#helm repo add stable https://charts.helm.sh/stable

echo 'export KUBECONFIG=/etc/rancher/k3s/k3s.yaml' >> ~/.bashrc

```

環境変数に設定するだけではなく`--kubeconfig $KUBECONFIG`のように書かないといけないので`.bashrc`にこれ書いておくとよい。

```
h () {
  sudo helm "$@" --kubeconfig=$KUBECONFIG
}
```

# 調べること
- コンパートメント
- VNIC
    - instanceごとにあるようだ

# 試していること
- iptablesの設定勉強した方がよさそう
- まずはk3s以外でポートがちゃんと開放されるか確認する

# memo
- k3sではDNSがうまく動作しない[問題](https://github.com/rancher/k3s/issues/1527)があるようだ
    - いや、以下のコマンドをインストール前に実行することで解決した。
    - 参考にしたのは[ここ](https://atelierhsn.com/2020/01/k3s-on-oracle-cloud/)
    - 永続化は[ここ](https://qiita.com/yas-nyan/items/e5500cf67236d11cce72)

```
# firewall setting
sudo iptables -I FORWARD -s 10.0.0.0/8 -j ACCEPT
sudo iptables -I FORWARD -d 10.0.0.0/8 -j ACCEPT
sudo iptables -I INPUT   -s 10.0.0.0/8 -j ACCEPT
sudo iptables -I INPUT   -d 10.0.0.0/8 -j ACCEPT
sudo /etc/init.d/netfilter-persistent save
sudo /etc/init.d/netfilter-persistent reload

# install k3s
curl -sfL https://get.k3s.io | sh -s - --write-kubeconfig-mode 640

# create k3s group
sudo groupadd k3s
sudo usermod -aG k3s `whoami`
sudo chgrp k3s /etc/rancher/k3s/k3s.yaml
```

- cert-managerも使いたい
    - やるなら[ここ](https://opensource.com/article/20/3/ssl-letsencrypt-k3s)参考。

- 公開方法の選択肢は[ここ](https://www.thebookofjoel.com/bare-metal-kubernetes-ingress)が詳しい。

- user group
    - 再起動の度に`chmod`が初期化されるらしい。`chgrp`は初期化されない
    - `--write-kubeconfig-mode 640`
    - `sudo`なしだと`kubectl get all`で問題が生じるが、[ここ](https://github.com/kubernetes/kubernetes/issues/94362)によると問題なさそう。
      進歩
- CI/CD
    - k3s側でGithubを監視する方法、GithubActionsからk3sにアクセスする方法の2通り考えられる
    - 前者を実現するためにArgoCDなどのツールがあるが、メモリが厳しい
    - 後者は[これ](https://github.com/rancher/k3s/issues/1381)によるとIPアドレスを指定して許可する必要があるが、
    GithubActions側のIPアドレスが分からないので難しそう
