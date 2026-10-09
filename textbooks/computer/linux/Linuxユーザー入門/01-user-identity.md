# 1. ユーザーの実体

## 1.1 カーネルが知っているのは数値だけ

カーネルの中では、ユーザーは UID（user ID）という整数で、グループは GID（group ID）という整数で表される。`alice` という名前はカーネルのどこにもない。

| 場所 | 持っているもの |
|---|---|
| プロセス（カーネル内の資格情報） | UID と GID の数値（[2章](02-process-credentials.md)） |
| ファイル（inode） | 所有者の UID と GID の数値、モードのビット（[3章](03-file-permission-check.md)） |
| /etc/passwd, /etc/group | 名前と数値の対応表。ただのテキストファイルで、カーネルは読まない |

名前と数値の変換は、ユーザー空間のライブラリ（libc）が行う。`getpwuid()` は UID から名前を、`getpwnam()` は名前から UID を引く関数である。どこから引くかは `/etc/nsswitch.conf` で決まる。この仕組みを NSS（Name Service Switch）と呼ぶ。

```
プロセス --getuid()--> カーネル : 1000 を返す
プロセス --getpwuid(1000)--> libc --(nsswitch.conf を見る)--> /etc/passwd を読む : "alice" を返す
```

1つ目はシステムコール、2つ目はライブラリ関数で、ファイルを読むだけである。

## 1.2 id コマンドが実際にしていること

```
alice$ id
uid=1000(alice) gid=1000(alice) groups=1000(alice),1003(dev)
```

`id` がどこから数値を得ているかを strace で見る。

```
alice$ strace -e trace=getuid,geteuid,getgid,getegid,getgroups id
geteuid()                               = 1000
getuid()                                = 1000
getegid()                               = 1000
getgid()                                = 1000
getgroups(0, NULL)                      = 2
getgroups(2, [1000, 1003])              = 2
uid=1000(alice) gid=1000(alice) groups=1000(alice),1003(dev)
+++ exited with 0 +++
```

カーネルが返したのは `1000` や `[1000, 1003]` という数値だけである。`getgroups(0, NULL)` は「いくつあるか」だけを聞く呼び方で、次の呼び出しで中身を受け取っている。

では、名前の `alice` や `dev` はどこから来たか。ファイルを開くシステムコール `openat` を見る。

```
alice$ strace -e trace=openat id
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/lib/aarch64-linux-gnu/libselinux.so.1", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/lib/aarch64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/lib/aarch64-linux-gnu/libpcre2-8.so.0", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/proc/filesystems", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/etc/nsswitch.conf", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/etc/passwd", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/etc/group", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/proc/sys/kernel/ngroups_max", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/proc/sys/kernel/ngroups_max", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/etc/group", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/etc/group", O_RDONLY|O_CLOEXEC) = 3
uid=1000(alice) gid=1000(alice) groups=1000(alice),1003(dev)
+++ exited with 0 +++
```

- 最初の4行は、共有ライブラリ（libc など）を読み込んでいるところである。
- `/etc/nsswitch.conf` を開いてから `/etc/passwd` と `/etc/group` を開いている。libc が NSS の設定を読み、その設定に従って対応表を読んだ。

この環境の設定は次のとおりで、ファイル（`files`）だけから引く。

```
alice$ grep -E "^(passwd|group):" /etc/nsswitch.conf
passwd:         files
group:          files
```

会社のサーバーなどでは、ここに `sss` や `ldap` が並び、名前をネットワークの先のディレクトリサービスから引くことがある。その場合も、カーネルから見ればプロセスの UID は数値のままである。

対応表の中身は次のとおりである。`/etc/passwd` の3番目と4番目の欄が UID と主グループの GID、`/etc/group` の3番目の欄が GID、4番目が補助グループとしてのメンバーである。

```
alice$ grep -E "^alice:" /etc/passwd
alice:x:1000:1000::/home/alice:/bin/bash
alice$ grep dev /etc/group
plugdev:x:46:
dev:x:1003:alice,bob
```

## 1.3 ファイルの所有者も数値

ファイルの所有者も、inode に記録された数値である。`ls -l` は名前に変換して表示し、`ls -ln` は変換せずに数値のまま表示する。

```
# cd /tmp && touch a.txt && chown alice:dev a.txt
# ls -l a.txt
-rw-r--r-- 1 alice dev  0 Oct  4 07:42 a.txt
# ls -ln a.txt
-rw-r--r-- 1 1000 1003 0 Oct  4 07:42 a.txt
```

カーネルが `ls` に返しているのが数値であることは、`statx` システムコールの結果で確かめられる（`-v` で構造体の中身を省略せずに出す）。

```
# strace -v -e trace=statx ls -l /tmp/a.txt 2>&1 | grep -o "stx_uid=[0-9]*, stx_gid=[0-9]*"
stx_uid=1000, stx_gid=1003
```

`ls -l` はこの数値を受け取ってから、1.2 と同じように `/etc/passwd` と `/etc/group` を開いて名前に変える。

## 1.4 対応表にない UID

対応表に載っていない UID を所有者にすることもできる。カーネルは数値を inode に書くだけで、その UID のユーザーが存在するかを確かめない。

```
# touch ghost.txt && chown 4242:4242 ghost.txt
# ls -l a.txt ghost.txt
-rw-r--r-- 1 alice dev  0 Oct  4 07:42 a.txt
-rw-r--r-- 1  4242 4242 0 Oct  4 07:42 ghost.txt
# getent passwd 4242; echo "rc=$?"
rc=2
```

`ls -l` は名前を引けなかったので、数値をそのまま表示している。`getent` は NSS 経由で対応表を引くコマンドで、終了コード 2 は「見つからない」を意味する。

ユーザーを削除したときも同じことが起きる。`userdel` は対応表から行を消すだけで、そのユーザーのファイルの inode は書き換えない（`-r` を付けなければホームディレクトリも残る）。

```
# userdel bob
# ls -l /home
total 12
drwxr-xr-x 2 alice  alice  4096 Oct  4 07:42 alice
drwxr-xr-x 2   1001   1001 4096 Oct  4 07:42 bob
drwxr-xr-x 2 nakano nakano 4096 Oct  4 07:42 nakano
```

同じ UID で対応表に行を足すと、そのディレクトリは再び `bob` のものとして表示される。ディレクトリ自体には何も起きていない。

```
# useradd -u 1001 -s /bin/bash bob && usermod -aG dev bob
# ls -l /home
total 12
drwxr-xr-x 2 alice  alice  4096 Oct  4 07:42 alice
drwxr-xr-x 2 bob    bob    4096 Oct  4 07:42 bob
drwxr-xr-x 2 nakano nakano 4096 Oct  4 07:42 nakano
```

逆に言えば、削除したユーザーの UID を別の人に割り当てると、その人が古いファイルの所有者になる。権限の判定（[3章](03-file-permission-check.md)）は数値で行われるからである。

## 1.5 まとめ

| 問い | 答え |
|---|---|
| カーネルはユーザーを何で区別するか | UID / GID の数値 |
| 名前はどこから来るか | libc が NSS の設定に従って /etc/passwd などから引く |
| `ls -l` の所有者名は inode に書いてあるか | 書いていない。数値を表示時に名前に変えている |
| ユーザーを消すとファイルはどうなるか | 何も変わらない。数値の持ち主が対応表にいなくなるだけ |
