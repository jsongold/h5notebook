# 練習問題

目的：各章の内容を、出力を予想してから実行して確かめる形で身につける。

- 環境は [README](../README.md) の「環境の作り方」で作ったコンテナを使う。4章・5章の問題は、各章で作ったプログラム（`showid`、`dropuid` など）を使う。
- 解答の出力は、本書の環境で実際に取ったものである。PID と時刻は実行のたびに変わる。
- 先に答えを書き出してから実行し、違ったら該当する章に戻る。

## 1章 ユーザーの実体

### 1-1 持ち主のいない UID に名前を付ける

```
# cd /tmp
# touch f5000 && chown 5000:5000 f5000
# ls -l f5000
# useradd -u 5000 carol
# ls -l f5000
# userdel carol
```

2回の `ls -l` で、所有者とグループの欄はそれぞれどう表示されるか。2回の間にファイルは変更されたか。

<details><summary>解答</summary>

```
-rw-r--r-- 1 5000 5000 0 Oct  4 07:46 f5000
-rw-r--r-- 1 carol carol 0 Oct  4 07:46 f5000
```

1回目は対応表に UID 5000 がないので数値のまま、2回目は `/etc/passwd` と `/etc/group` に carol が加わったので名前になる（`useradd` は同じ番号のグループ carol も作った）。ファイルの inode は一度も変更されていない。変わったのは `ls` が引く対応表だけである（[1章](../01-user-identity.md) 1.4）。

</details>

### 1-2 ls -ln は /etc/passwd を開くか

```
# strace -e trace=openat ls -ln /tmp/a.txt 2>&1 | grep -E "passwd|group|a.txt"
```

`/etc/passwd` や `/etc/group` を開く行は出るか。

<details><summary>解答</summary>

```
-rw-r--r-- 1 1000 1003 0 Oct  4 07:42 /tmp/a.txt
```

出るのは `ls` 自身の出力の1行だけで、`/etc/passwd` も `/etc/group` も開いていない。`-n` は数値のまま表示するので、名前に変える必要がない。比べて、`ls -l` では両方を開く（[1章](../01-user-identity.md) 1.3）。

</details>

### 1-3 ユーザー名を変える

```
# usermod -l alicia alice
# ls -l /tmp/a.txt
# ls -ln /tmp/a.txt
# grep ^alicia /etc/passwd
# usermod -l alice alicia
```

`ls -l` と `ls -ln` の所有者の欄はどうなるか。

<details><summary>解答</summary>

```
-rw-r--r-- 1 alicia dev 0 Oct  4 07:42 /tmp/a.txt
-rw-r--r-- 1 1000 1003 0 Oct  4 07:42 /tmp/a.txt
alicia:x:1000:1000::/home/alice:/bin/bash
```

`usermod -l` は `/etc/passwd` の1番目の欄（名前）を書き換えるだけで、UID 1000 は変わらない。そのため、ファイルに触れなくても `ls -l` の表示は `alicia` に変わり、`ls -ln` は 1000 のままである。ホームディレクトリのパスも `/home/alice` のまま残っている。最後のコマンドで名前を戻しておく。

</details>

## 2章 プロセスが持つ資格情報

### 2-1 他のプロセスの資格情報を読む

```
alice$ grep -E "^(Uid|Gid)" /proc/1/status /proc/self/status
```

PID 1（コンテナの `sleep infinity`）と、`grep` 自身の Uid / Gid はどう出るか。そもそも alice は PID 1 の status を読めるか。

<details><summary>解答</summary>

```
/proc/1/status:Uid:	0	0	0	0
/proc/1/status:Gid:	0	0	0	0
/proc/self/status:Uid:	1000	1000	1000	1000
/proc/self/status:Gid:	1000	1000	1000	1000
```

`/proc/<PID>/status` は誰でも読める。PID 1 は root で動いているので 4つとも 0、`grep` は alice のシェルから起動されたので 4つとも 1000 である（[2章](../02-process-credentials.md) 2.3）。

</details>

### 2-2 sudo と子プロセスの RUSER / USER

```
alice$ sudo -u bob sleep 20 &
alice$ ps -o pid,ppid,ruser,user,cmd -C sudo,sleep --forest
```

`sudo` の行と `sleep 20` の行で、`RUSER`（実 UID）と `USER`（実効 UID）はそれぞれ誰になるか。

<details><summary>解答</summary>

```
    PID    PPID RUSER    USER     CMD
     15      14 alice    root     sudo -u bob sleep 20
     17      15 bob      bob       \_ sleep 20
      1       0 root     root     sleep infinity
```

`sudo` は alice が set-user-ID root のファイルを実行したものなので、実は alice、実効は root である。`sleep` は sudo の子が `setresuid` で3つとも bob にしてから execve したものなので、両方 bob になる（[6章](../06-sudo-su.md) 6.2）。最後の行は PID 1 の `sleep infinity` が `-C sleep` に当てはまったものである。

</details>

### 2-3 グループに追加しても、動いているプロセスは変わらない

```
# groupadd staff2
alice$ sleep 30 &                       # この PID を P とする
# usermod -aG staff2 alice
# grep Groups /proc/P/status            # P は上の sleep の PID
alice$ grep Groups /proc/self/status    # 新しく開いた alice のシェルから
# getent group staff2
```

`usermod` の前から動いている `sleep` と、`usermod` の後に新しく始めた alice のプロセスで、`Groups` はどう違うか。

<details><summary>解答</summary>

```
Groups:	1000 1003 
Groups:	1000 1003 1004 
staff2:x:1004:alice
```

`usermod -aG` が書き換えるのは `/etc/group` だけである。補助グループはプロセスの cred の属性で、ログイン（su、sudo など）のときに `setgroups` で設定され、あとは fork で引き継がれるだけである（[2章](../02-process-credentials.md) 2.2）。そのため、前から動いている `sleep` には 1004 がなく、新しくログインした（`su alice` で作った）プロセスにはある。グループを追加したら「ログインし直す」必要があるのはこのためである。後始末は `gpasswd -d alice staff2; groupdel staff2`。

</details>

## 3章 ファイルを開くとき誰が何を判定するか

### 3-1 誰が書き込めるか

```
alice$ cd /srv/lab
alice$ echo x > p1.txt; chgrp dev p1.txt; chmod 464 p1.txt; ls -l p1.txt
alice$ echo alice >> /srv/lab/p1.txt && echo alice: ok
bob$ echo bob >> /srv/lab/p1.txt && echo bob: ok
nakano$ echo nakano >> /srv/lab/p1.txt && echo nakano: ok
```

alice（所有者）、bob（dev のメンバー）、nakano（その他）のうち、追記できるのは誰か。

<details><summary>解答</summary>

```
-r--rw-r-- 1 alice dev 2 Oct  4 07:47 p1.txt
bash: line 1: /srv/lab/p1.txt: Permission denied
bob: ok
bash: line 1: /srv/lab/p1.txt: Permission denied
```

1行目の拒否が alice、最後の拒否が nakano である。alice は所有者なので1組目の `r--` だけで判定され、グループの `rw-` は見られない。bob はグループの `rw-`、nakano はその他の `r--` で判定される（[3章](../03-file-permission-check.md) 3.2）。

</details>

### 3-2 共有ディレクトリとスティッキービット

```
alice$ cd /srv/lab
alice$ mkdir shared; chgrp dev shared; chmod 770 shared
alice$ echo a > shared/f; chmod 600 shared/f
bob$ rm -f /srv/lab/shared/f && echo bob removed f
alice$ echo a > shared/f; chmod 600 shared/f; chmod +t shared; ls -ld shared
bob$ rm -f /srv/lab/shared/f && echo bob removed f
```

bob は、自分が読むこともできない alice のファイル `f` を消せるか。スティッキービット（`+t`）を付けた後はどうか。

<details><summary>解答</summary>

```
bob removed f
drwxrwx--T 2 alice dev 4096 Oct  4 07:47 shared
rm: cannot remove '/srv/lab/shared/f': Operation not permitted
```

1回目は消せる。削除はディレクトリの表を書き換える操作で、bob はグループとしてディレクトリの `rwx` を持っているからである（[3章](../03-file-permission-check.md) 3.3）。スティッキービットを付けると、`man 7 inode` のとおり、削除できるのはファイルの所有者、ディレクトリの所有者、特権のあるプロセスだけになり、bob は `EPERM`（Operation not permitted）になる。`ls -ld` の最後の `T` は、スティッキービットがあり、その他の x がないことを表す。

</details>

### 3-3 ケーパビリティのない root

```
# cd /srv/lab
# echo root > rootonly.txt; chmod 000 rootonly.txt; ls -l rootonly.txt
# cat rootonly.txt
# capsh --drop=cap_dac_override -- -c "cat /srv/lab/rootonly.txt"
```

root が所有者で、モードは 000。2回の `cat` はそれぞれどうなるか。

<details><summary>解答</summary>

```
---------- 1 root root 5 Oct  4 07:47 rootonly.txt
root
cat: /srv/lab/rootonly.txt: Permission denied
```

1回目は `CAP_DAC_OVERRIDE` で判定を通り抜けて読める。2回目は UID 0 で所有者に当てはまるが、所有者のビットが `---` なので拒否され、それを覆すケーパビリティもない（Docker の root は `CAP_DAC_READ_SEARCH` も持っていない）。root であっても、判定そのものは普通のユーザーと同じ順番で行われている（[3章](../03-file-permission-check.md) 3.4）。

</details>

## 4章 fork / execve での引き継ぎ

### 4-1 bob が set-user-ID root のプログラムを実行する

```
bob$ showid-suid
```

`uid(r,e,s)` と `gid(r,e,s)` と `groups` はどうなるか。

<details><summary>解答</summary>

```
showid pid=94 ppid=93 uid(r,e,s)=(1001,0,0) gid(r,e,s)=(1001,1001,1001) groups=1001,1003
```

実 UID は bob（1001）のまま、実効と保存がファイルの所有者 root（0）になる。GID と補助グループは変わらない（[4章](../04-fork-execve.md) 4.4）。

</details>

### 4-2 root 以外が所有する set-user-ID

```
# cp /usr/local/bin/showid /usr/local/bin/showid-alice
# chown alice /usr/local/bin/showid-alice
# chmod 4755 /usr/local/bin/showid-alice
# ls -l /usr/local/bin/showid-alice
bob$ showid-alice
```

bob が実行したとき、3つの UID はどうなるか。

<details><summary>解答</summary>

```
-rwsr-xr-x 1 alice root 70760 Oct  4 07:47 /usr/local/bin/showid-alice
showid pid=100 ppid=99 uid(r,e,s)=(1001,1000,1000) gid(r,e,s)=(1001,1001,1001) groups=1001,1003
```

set-user-ID は「root になる」仕組みではなく、「実効 UID をファイルの所有者にする」仕組みである。所有者が alice なら、実効と保存が 1000 になる（[4章](../04-fork-execve.md) 4.4）。`chown` を先に、`chmod` を後にしているのは、実行ファイルの所有者を変えると set-user-ID ビットが外されるからである（`man 2 chown`。この環境でも、`chmod 4755` のあとに `chown alice` すると `-rwsr-xr-x` が `-rwxr-xr-x` になった）。

</details>

### 4-3 子が UID を変えても親は変わらない

```
# cat > /tmp/p43.py <<'EOF'
import os
pid = os.fork()
if pid == 0:
    os.setuid(1001)
    print("child ", os.getpid(), os.getresuid(), flush=True)
    os._exit(0)
os.waitpid(pid, 0)
print("parent", os.getpid(), os.getresuid(), flush=True)
EOF
# python3 /tmp/p43.py
```

子が `setuid(1001)` したあと、親の `getresuid()` は何を返すか。

<details><summary>解答</summary>

```
child  122 (1001, 1001, 1001)
parent 121 (0, 0, 0)
```

fork で子は親の cred のコピーを持つ（[4章](../04-fork-execve.md) 4.1）。子が `setuid` で書き換えるのは自分のコピーだけで、親の cred は別物なので変わらない。root の子が `setuid(1001)` を呼んだので、子は3つとも 1001 になっている（[5章](../05-switching-uid.md) 5.1）。

</details>

## 5章 権限を切り替えるシステムコール

### 5-1 bob が dropuid-suid を実行する

```
bob$ dropuid-suid
```

5行の出力を予想する（[5章](../05-switching-uid.md) 5.2 の alice の結果を、bob に置き換えて考える）。

<details><summary>解答</summary>

```
start                    uid(r,e,s)=(1001,0,0)
seteuid(me)              uid(r,e,s)=(1001,1001,0)
seteuid(0)               uid(r,e,s)=(1001,0,0)
setresuid(me,me,me)      uid(r,e,s)=(1001,1001,1001)
seteuid(0)               -> Operation not permitted
```

保存 UID に 0 が残っている間は `seteuid(0)` で戻れる。`setresuid(1001,1001,1001)` で 0 がどこにもなくなると、もう戻れない。

</details>

### 5-2 順番を間違えた root

```
# python3 -c "import os; os.setresuid(1001,1001,1001); os.setgroups([])"
```

UID を先に変えてから補助グループを空にしようとしている。どうなるか。

<details><summary>解答</summary>

```
Traceback (most recent call last):
  File "<string>", line 1, in <module>
PermissionError: [Errno 1] Operation not permitted
```

`setresuid` で3つの UID がすべて 0 以外になった時点で、カーネルが permitted と effective のケーパビリティを消す（`man 7 capabilities`）。`setgroups` には `CAP_SETGID` が要るので失敗する。このプロセスは root の補助グループ（ここでは 0）を持ったまま bob として動き続けることになり、危険である。だから順番は setgroups → setresgid → setresuid になる（[5章](../05-switching-uid.md) 5.3）。

</details>

### 5-3 alice 所有の set-user-ID で、0 に戻ろうとする

```
# cp /usr/local/bin/dropuid /usr/local/bin/dropuid-alice
# chown alice /usr/local/bin/dropuid-alice
# chmod 4755 /usr/local/bin/dropuid-alice
bob$ dropuid-alice
```

5行の出力を予想する。

<details><summary>解答</summary>

```
start                    uid(r,e,s)=(1001,1000,1000)
seteuid(me)              uid(r,e,s)=(1001,1001,1000)
seteuid(0)               -> Operation not permitted
setresuid(me,me,me)      uid(r,e,s)=(1001,1001,1001)
seteuid(0)               -> Operation not permitted
```

このプロセスが持っている値は 1001 と 1000 だけで、0 は一度も持っていない。特権のないプロセスは、今持っている値の間でしか入れ替えられないので、`seteuid(0)` は最初から失敗する。一方、`seteuid(1001)` は実 UID の値なので成功する。

</details>

## 6章 sudo / su の中で起きていること

### 6-1 カレントディレクトリ

```
alice$ cd /srv/lab
alice$ pwd
alice$ sudo -u nakano pwd
alice$ sudo -u nakano -i pwd
```

3行の出力を予想する。

<details><summary>解答</summary>

```
/srv/lab
/srv/lab
/home/nakano
```

カレントディレクトリはプロセスの属性で、fork と execve で引き継がれる。`-u` だけでは誰も `chdir` しないので alice のディレクトリのまま、`-i` では sudo がシェルを起動する前に `chdir` で対象ユーザーのホームに移る（[6章](../06-sudo-su.md) 6.4）。

</details>

### 6-2 $HOME は誰が展開するか

```
alice$ sudo -u nakano -i echo $HOME
alice$ sudo -u nakano -i echo '$HOME'
alice$ sudo -u nakano -i 'echo $HOME'
```

3つの出力を予想する。ヒント：`man sudo` によると、`-i` はコマンドの文字をエスケープするが、英数字・`_`・`-`・`$` はエスケープしない。

<details><summary>解答</summary>

```
/home/alice
/home/nakano
-bash: line 1: echo /home/nakano: No such file or directory
```

- 1つ目は、alice のシェルが先に `$HOME` を `/home/alice` に展開してから sudo に渡す。
- 2つ目は、`$HOME` がそのまま sudo に届く。`$` はエスケープされないので、nakano のログインシェルが `echo $HOME` を解釈して `/home/nakano` になる。
- 3つ目は、引数が1つなので空白がエスケープされ、`echo\ $HOME` が渡る。nakano のシェルは `$HOME` だけを展開し、`echo /home/nakano` という1つの名前のコマンドを探す。名前に `/` が含まれるのでパスとして扱われ、そのファイルがないので `No such file or directory` になる。

strace で bash に渡った引数を見ると、2つ目と3つ目の違いがわかる。

```
# strace -f -qq -s 200 -o /tmp/p62.trace -e trace=execve -u alice sudo -u nakano -i 'echo $HOME'
# grep bash /tmp/p62.trace
157   execve("/bin/bash", ["-bash", "--login", "-c", "echo\\ $HOME"], 0xaaaad07c3cf0 /* 13 vars */) = 0
# strace -f -qq -s 200 -o /tmp/p62b.trace -e trace=execve -u alice sudo -u nakano -i echo '$HOME'
# grep bash /tmp/p62b.trace
164   execve("/bin/bash", ["-bash", "--login", "-c", "echo $HOME"], 0xaaaae45f0cf0 /* 13 vars */) = 0
```

</details>

### 6-3 補助グループは誰のものになるか

```
alice$ id
alice$ sudo -u bob id
alice$ sudo -u nakano id
```

alice の補助グループ dev（1003）は、sudo の先のプロセスに残るか。

<details><summary>解答</summary>

```
uid=1000(alice) gid=1000(alice) groups=1000(alice),1003(dev)
uid=1001(bob) gid=1001(bob) groups=1001(bob),1003(dev)
uid=1002(nakano) gid=1002(nakano) groups=1002(nakano)
```

残らない。sudo は fork の前に `setgroups` で補助グループを対象ユーザーのもの（`/etc/group` から引いたもの）に置き換える（[6章](../06-sudo-su.md) 6.2）。bob に dev があるのは bob 自身が dev のメンバーだからで、nakano にはない。

</details>
