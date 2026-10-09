# 3. ファイルを開くとき誰が何を判定するか

## 3.1 判定するのはカーネル

プロセスが `open()`（実際のシステムコールは `openat`）を呼ぶと、カーネルは次の2つを突き合わせて、許すか `EACCES` を返すかを決める。

```
プロセスの cred（fsuid, fsgid, 補助グループ, ケーパビリティ）
        ×
ファイルの inode（所有者 UID, グループ GID, モードのビット）
```

プログラムの側は何も判定していない。`cat` は `openat` を呼び、カーネルが返したエラーを表示するだけである。

実験用に、誰でも書ける作業ディレクトリ `/srv/lab` を作っておく。

```
# mkdir -p /srv/lab && chmod 777 /srv/lab
alice$ cd /srv/lab
alice$ echo secret > memo.txt
alice$ chmod 600 memo.txt
alice$ stat memo.txt
  File: memo.txt
  Size: 7         	Blocks: 8          IO Block: 4096   regular file
Device: 0,55	Inode: 3340495     Links: 1
Access: (0600/-rw-------)  Uid: ( 1000/   alice)   Gid: ( 1000/   alice)
Access: 2026-10-04 07:42:57.345337007 +0000
Modify: 2026-10-04 07:42:57.346394423 +0000
Change: 2026-10-04 07:42:57.346720548 +0000
 Birth: 2026-10-04 07:42:57.345337007 +0000
```

`stat` の `Access:` の行が、判定に使われる inode の値である。モード `0600`、所有者 UID 1000、グループ GID 1000。

bob（UID 1001）がこのファイルを読もうとすると、次のようになる。

```
bob$ strace -e trace=openat cat memo.txt 2>&1 | tail -4
openat(AT_FDCWD, "/lib/aarch64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "memo.txt", O_RDONLY)  = -1 EACCES (Permission denied)
cat: memo.txt: Permission denied
+++ exited with 1 +++
```

`openat` が `-1 EACCES` を返し、`cat` はそれを `Permission denied` と表示した。ファイルの中身には一度も触れていない。

## 3.2 判定の順番：最初に当てはまった1組だけを見る

`man 7 path_resolution` の Permissions の節には、次のように書かれている。

1. プロセスの実効 UID（Linux では fsuid）がファイルの所有者と等しければ、最初の3ビット（所有者の rwx）を使う。
2. そうでなく、ファイルのグループが実効 GID（fsgid）か補助グループのどれかと等しければ、次の3ビット（グループの rwx）を使う。
3. どちらでもなければ、最後の3ビット（その他の rwx）を使う。

カーネルの実装は `fs/namei.c` の `acl_permission_check()` で、ACL を除いて書くと次の擬似コードになる。

```
if (inode の所有者 == fsuid)            → 所有者のビットだけで判定して終わり
else if (inode のグループ ∈ {fsgid, 補助グループ}) → グループのビットだけで判定して終わり
else                                     → その他のビットで判定
```

大事なのは、当てはまった1組だけを見て終わる点である。「どれか1組で許されればよい」のではない。そのため、所有者がその他より弱い権限しか持たない、ということが起こる。

```
alice$ echo hello > trap.txt
alice$ chmod 077 trap.txt
alice$ ls -l trap.txt
----rwxrwx 1 alice alice 6 Oct  4 07:43 trap.txt
alice$ cat trap.txt
cat: trap.txt: Permission denied
bob$ cat /srv/lab/trap.txt
hello
```

alice は所有者なので、1組目（`---`）だけで判定されて拒否される。bob はその他に当たり、`rwx` で許される。

グループでも同じことが起きる。

```
alice$ echo team > g.txt
alice$ chgrp dev g.txt
alice$ chmod 604 g.txt
alice$ ls -l g.txt
-rw----r-- 1 alice dev 5 Oct  4 07:43 g.txt
bob$ cat /srv/lab/g.txt
cat: /srv/lab/g.txt: Permission denied
nakano$ cat /srv/lab/g.txt
team
```

| ユーザー | 当てはまる組 | 使うビット | 読めるか |
|---|---|---|---|
| alice | 所有者 | `rw-` | 読める |
| bob（dev のメンバー） | グループ | `---` | 読めない |
| nakano（dev ではない） | その他 | `r--` | 読める |

ただし、モードを変えられるかどうかは権限のビットではなく、「実効 UID がファイルの所有者と等しいか（または `CAP_FOWNER` を持つか）」で決まる（`man 2 chmod`）。そのため alice は `---` にした自分のファイルでも、いつでも `chmod` で戻せる。

```
alice$ chmod 000 trap.txt; ls -l trap.txt; chmod 644 trap.txt; ls -l trap.txt
---------- 1 alice alice 6 Oct  4 07:43 trap.txt
-rw-r--r-- 1 alice alice 6 Oct  4 07:43 trap.txt
```

## 3.3 ディレクトリの r / w / x

ディレクトリの中身は「名前 → inode 番号」の表である。3つのビットはこの表に対する操作を表す。

| ビット | 許すこと |
|---|---|
| r | 表の名前の一覧を読む（`ls`） |
| w | 表に名前を足す・消す・変える（作成、削除、名前の変更）。x も必要 |
| x | 表を通り抜けて、名前から inode をたどる（検索。`man 7 path_resolution` では search permission） |

パス `/srv/lab/d_x/f` を開くとき、カーネルは `/`、`/srv`、`/srv/lab`、`/srv/lab/d_x` のそれぞれで x を確認してから `f` にたどり着く。

alice が3つのディレクトリを作り、bob から試す。

```
alice$ mkdir d_r d_x d_wx
alice$ echo a > d_r/f; echo b > d_x/f; echo c > d_wx/f
alice$ chmod 744 d_r; chmod 711 d_x; chmod 733 d_wx; chmod 600 d_wx/f
alice$ ls -ld d_r d_x d_wx
drwxr--r-- 2 alice alice 4096 Oct  4 07:43 d_r
drwx-wx-wx 2 alice alice 4096 Oct  4 07:43 d_wx
drwx--x--x 2 alice alice 4096 Oct  4 07:43 d_x
```

r だけ（`d_r`）では、名前は読めるが、その先の inode をたどれない。

```
bob$ ls /srv/lab/d_r
f
bob$ ls -l /srv/lab/d_r
ls: cannot access '/srv/lab/d_r/f': Permission denied
total 0
-????????? ? ? ? ?            ? f
bob$ cat /srv/lab/d_r/f
cat: /srv/lab/d_r/f: Permission denied
```

`ls -l` は名前 `f` を得たあと、`f` の inode を `statx` で調べようとして失敗した。そのため、名前以外がすべて `?` になっている。

x だけ（`d_x`）では、一覧は読めないが、名前を知っていればたどれる。

```
bob$ ls /srv/lab/d_x
ls: cannot open directory '/srv/lab/d_x': Permission denied
bob$ cat /srv/lab/d_x/f
b
```

w と x（`d_wx`）があれば、読めないファイルでも削除できる。削除はファイルではなく、ディレクトリの表を書き換える操作だからである。

```
bob$ cat /srv/lab/d_wx/f
cat: /srv/lab/d_wx/f: Permission denied
bob$ rm -f /srv/lab/d_wx/f && echo removed
removed
bob$ touch /srv/lab/d_wx/bobfile && ls -l /srv/lab/d_wx/bobfile
-rw-r--r-- 1 bob bob 0 Oct  4 07:43 /srv/lab/d_wx/bobfile
```

他人のファイルを消されたくない共有ディレクトリ（`/tmp` など）では、スティッキービットを付ける。`man 7 inode` によると、スティッキービットのあるディレクトリでは、ファイルの名前の変更と削除ができるのは、そのファイルの所有者、ディレクトリの所有者、特権を持つプロセスだけになる（[練習問題](practice/problems.md) 3-2）。

## 3.4 root はなぜ読めるのか：ケーパビリティ

root（UID 0）が判定を通り抜けるのは、UID が 0 だからではなく、ケーパビリティ（capability）を持っているからである。`man 7 path_resolution` によると、ファイルの判定に関わるのは次の2つである。

| ケーパビリティ | 効果 |
|---|---|
| `CAP_DAC_OVERRIDE` | すべての権限の判定を通り抜ける。ただし実行は、3組のどれかに x があるときだけ許す |
| `CAP_DAC_READ_SEARCH` | ファイルの読み取りと、ディレクトリの読み取り・検索を許す |

カーネルでは、`fs/namei.c` の `generic_permission()` が、まず 3.2 の判定を行い、拒否されたときにこれらのケーパビリティを確かめる。

Docker のコンテナの root が持っているケーパビリティを見る。

```
# grep CapEff /proc/self/status
CapEff:	00000000a80c25fb
# capsh --decode=00000000a80c25fb
0x00000000a80c25fb=cap_chown,cap_dac_override,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_net_raw,cap_sys_chroot,cap_sys_ptrace,cap_mknod,cap_audit_write,cap_setfcap
```

`cap_dac_override` はあるが、`cap_dac_read_search` は Docker の既定では持っていない（`cap_sys_ptrace` は本書の環境で足したもの）。

モード 000 のファイルと、x のないスクリプトを alice が作り、root で試す。

```
alice$ echo zero > z.txt; chmod 000 z.txt
alice$ printf "#!/bin/sh\necho run\n" > s.sh; chmod 644 s.sh
# cd /srv/lab
# ls -l z.txt s.sh
-rw-r--r-- 1 alice alice 19 Oct  4 07:43 s.sh
---------- 1 alice alice  5 Oct  4 07:43 z.txt
# cat z.txt
zero
# ./s.sh; echo rc=$?
bash: line 7: ./s.sh: Permission denied
rc=126
```

- `z.txt` は 000 でも読める。`CAP_DAC_OVERRIDE` があるからである。
- `s.sh` は実行できない。3組のどれにも x がないので、`CAP_DAC_OVERRIDE` でも実行は許されない。

`capsh --drop` でケーパビリティを外した root で試す。

```
# capsh --drop=cap_dac_override,cap_dac_read_search -- -c "id -u; grep CapEff /proc/self/status; cat /srv/lab/z.txt"
0
CapEff:	00000000a80c25f9
cat: /srv/lab/z.txt: Permission denied
```

UID は 0 のままだが読めなくなった。`a80c25fb` から `a80c25f9` へ、`CAP_DAC_OVERRIDE`（ビット1）だけが消えている。権限の判定で root を特別扱いしているのは、UID の値ではなくケーパビリティである。

## 3.5 まとめ

| 問い | 答え |
|---|---|
| 判定するのは誰か | カーネル（`openat` などのシステムコールの中） |
| 何と何を比べるか | プロセスの fsuid / fsgid / 補助グループと、inode の所有者 / グループ / モード |
| 判定の順番 | 所有者 → グループ → その他。最初に当てはまった1組だけを見る |
| ファイルの削除に要るのは | ファイルの w ではなく、ディレクトリの w と x |
| root が通れる理由 | `CAP_DAC_OVERRIDE` などのケーパビリティ。UID 0 そのものではない |
