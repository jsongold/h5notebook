# Linux ユーザー入門

目的：Linux の「ユーザー」が、カーネルの中では何として存在し、どのプロセスがどのシステムコールでそれを変えるのかを理解する。コマンドとコードを実際に動かし、出力を見ながら確かめる。

## 目次

1. [ユーザーの実体](01-user-identity.md)：カーネルは数値の UID / GID しか知らない。名前は libc が /etc/passwd から引く
2. [プロセスが持つ資格情報](02-process-credentials.md)：`struct cred` の中身、実 / 実効 / 保存 / ファイルシステム UID と補助グループ
3. [ファイルを開くとき誰が何を判定するか](03-file-permission-check.md)：open() での owner → group → other の判定、ディレクトリの r / w / x、root とケーパビリティ
4. [fork / execve での引き継ぎ](04-fork-execve.md)：子は親の資格情報をコピーする。execve は set-user-ID ビットのときだけ変える
5. [権限を切り替えるシステムコール](05-switching-uid.md)：setuid / seteuid / setresuid / setgroups の規則、一時的に降りる・完全に降りる、ケーパビリティとの関係
6. [sudo / su の中で起きていること](06-sudo-su.md)：認証から setgroups → setgid → setresuid → execve まで、`-u` と `-i` の違い、プロセスの木
- [練習問題](practice/problems.md)：章ごとに3問。出力を予想してから実行して確かめる

## 前提

- 実行環境：macOS 上の Docker Desktop で動かした Linux コンテナ。イメージは `debian:bookworm-slim`（Debian 12.15）、カーネルは `uname -r` で `7.0.12-linuxkit`、アーキテクチャは aarch64。sudo 1.9.13p3、strace 6.1、Python 3.11.2。
- 出力はすべて 2026-10-04 にこの環境で実際に取ったものである。PID・時刻・ライブラリのパス（`/lib/aarch64-linux-gnu/...`）は環境によって変わる。
- システムコールと sudo の挙動は、コンテナ内の man ページ（`man 2 execve`、`man 2 setresuid`、`man 7 path_resolution`、`man 7 capabilities`、`man sudo`、`man sudoers`）と Linux カーネルのソース（`include/linux/cred.h`、`kernel/sys.c`、`fs/namei.c`）で確認した。
- Docker の root は、普通の Linux の root より持っているケーパビリティが少ない（3章、5章）。本書の root の結果はこの条件のものである。

### 環境の作り方

strace は ptrace を使うので、`SYS_PTRACE` を足し、seccomp を外して起動する。

```sh
docker run -d --name lxuser --hostname lab \
  --cap-add=SYS_PTRACE --security-opt seccomp=unconfined \
  debian:bookworm-slim sleep infinity
docker exec -it lxuser bash
```

コンテナの中（root）で次を実行する。`/etc/dpkg/dpkg.cfg.d/docker` は man ページを入れない設定なので、先に消す。

```sh
rm -f /etc/dpkg/dpkg.cfg.d/docker
apt-get update
apt-get install -y strace gcc libc6-dev sudo python3 python3-pip libcap2-bin procps man-db manpages manpages-dev
useradd -m -s /bin/bash alice     # uid 1000
useradd -m -s /bin/bash bob       # uid 1001
useradd -m -s /bin/bash nakano    # uid 1002
groupadd dev                      # gid 1003
usermod -aG dev alice
usermod -aG dev bob
echo "alice ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers.d/alice
chmod 440 /etc/sudoers.d/alice
su - nakano -c "pip install --user --break-system-packages pipenv"   # 6章で使う
```

### 表記

- `#` で始まる行は root で、`alice$`（`bob$`、`nakano$`）で始まる行はそのユーザーで実行したコマンドである。alice のシェルは `docker exec -it -u alice -w /home/alice lxuser bash` で開く。
- 出力は `docker exec ... bash -c '...'` や `su alice -c '...'` 経由で取ったので、bash のエラーメッセージに `line 1:` のような行番号が付いている。対話シェルで打つと行番号は付かない。
- 出力の一部だけを見せるときは、`grep` などで絞ったことがわかるようにコマンドに含めるか、「省略」と書く。
- コンテナの PID 1 は `sleep infinity` なので、`ps` の出力に出てくる。
