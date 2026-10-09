# 2. プロセスが持つ資格情報

## 2.1 ユーザーはプロセスの属性である

「alice がログインしている」とは、カーネルから見れば「UID 1000 を持つプロセスがある」ということである。ユーザーはプロセスごとの属性で、カーネルはそれをプロセスの構造体（`task_struct`）から指される `struct cred` に入れている。

Linux のソース（`include/linux/cred.h`）から、ユーザーに関係する欄だけを抜き出すと次のようになる。

```c
struct cred {
	kuid_t		uid;		/* real UID of the task */
	kgid_t		gid;		/* real GID of the task */
	kuid_t		suid;		/* saved UID of the task */
	kgid_t		sgid;		/* saved GID of the task */
	kuid_t		euid;		/* effective UID of the task */
	kgid_t		egid;		/* effective GID of the task */
	kuid_t		fsuid;		/* UID for VFS ops */
	kgid_t		fsgid;		/* GID for VFS ops */
	kernel_cap_t	cap_permitted;	/* caps we're permitted */
	kernel_cap_t	cap_effective;	/* caps we can actually use */
	struct group_info *group_info;	/* supplementary groups for euid/fsgid */
	/* ほかの欄は省略 */
};
```

```
task_struct（プロセス） --cred--> struct cred { uid, euid, suid, fsuid, gid, ..., group_info, cap_* }
```

## 2.2 4種類の UID

1つのプロセスは UID を4つ持つ。GID も同じ4種類がある。

| 名前 | cred の欄 | 何に使うか |
|---|---|---|
| 実 UID（real） | `uid` | 「誰が起動したか」。シグナルを送れる相手の判定などに使う |
| 実効 UID（effective） | `euid` | 「今、誰として動いているか」。権限の判定に使う |
| 保存 UID（saved set-user-ID） | `suid` | 実効 UID を戻すための控え（[5章](05-switching-uid.md)） |
| ファイルシステム UID | `fsuid` | ファイルの権限判定に使う。普通は実効 UID と常に同じ値 |

普段は4つとも同じ値である。値が分かれるのは、set-user-ID ビット付きのプログラムを実行したとき（[4章](04-fork-execve.md)）や、プログラム自身が切り替えたとき（[5章](05-switching-uid.md)）である。

補助グループ（supplementary groups）は、主グループ（`gid`）とは別に持つグループの一覧である。`/etc/group` で alice が `dev` のメンバーになっていても、それがプロセスの補助グループに入るのはログインのときで、入れるのは login や sudo などのプログラムである（[6章](06-sudo-su.md)）。

## 2.3 /proc で見る

`/proc/<PID>/status` には、カーネルの cred の中身がそのまま出る。`man 5 proc` によると、`Uid` と `Gid` の行は「実、実効、保存、ファイルシステム」の順である。`/proc/self` は読んでいるプロセス自身を指す。

```
alice$ grep -E "^(Uid|Gid|Groups)" /proc/self/status
Uid:	1000	1000	1000	1000
Gid:	1000	1000	1000	1000
Groups:	1000 1003 
```

ここで `grep` が読んでいるのは `grep` 自身の資格情報である。alice のシェルから起動されたので、alice の値を受け継いでいる（[4章](04-fork-execve.md)）。

root のプロセスを読むと次のようになる。

```
# grep -E "^(Uid|Gid|Groups)" /proc/self/status
Uid:	0	0	0	0
Gid:	0	0	0	0
Groups:	0
```

## 2.4 ps で見る

`ps` の `-o` で、4種類の UID を列として出せる。`USER` 列は実効 UID を名前に直したものである。

```
alice$ sleep 300 &
alice$ ps -o pid,ruid,euid,suid,fsuid,user,cmd -p $$,$!
    PID  RUID  EUID  SUID FSUID USER     CMD
   4466  1000  1000  1000  1000 alice    bash
   4473  1000  1000  1000  1000 alice    sleep 300
```

（`CMD` 列は、実際には `bash -c` に渡したコマンド列全体が表示されたので、`bash` に縮めている。）

値が分かれている例として、パスワードを変える `passwd` を alice が実行している最中を見る。`passwd` は入力を待って止まるので、その間に別の端末から見られる。

```
alice$ ps -o pid,ruid,euid,suid,fsuid,user,ruser,cmd -C passwd
    PID  RUID  EUID  SUID FSUID USER     RUSER    CMD
   4493  1000     0     0     0 root     alice    passwd
alice$ grep -E "^(Uid|Gid)" /proc/4493/status
Uid:	1000	0	0	0
Gid:	1000	1000	1000	1000
```

- 実 UID は 1000（alice が起動した）、実効 UID は 0（root として動いている）である。
- `USER` 列は実効 UID なので `root`、`RUSER` 列は実 UID なので `alice` になる。
- GID は変わっていない。`passwd` は UID だけを変えるプログラムだからである（理由は[4章](04-fork-execve.md)の set-user-ID ビット）。

## 2.5 プログラムから見る

プログラムからは、`getresuid()` / `getresgid()` で3種類（実、実効、保存）を、`getgroups()` で補助グループを得る。Python では `os` モジュールに同じ名前の関数がある。

```
alice$ python3 -c "import os; print(os.getresuid(), os.getresgid(), os.getgroups())"
(1000, 1000, 1000) (1000, 1000, 1000) [1000, 1003]
```

ファイルシステム UID を返すシステムコールはない（`setfsuid()` の戻り値で間接的にわかるだけ）。見るなら `/proc/self/status` を読む。

## 2.6 まとめ

| 問い | 答え |
|---|---|
| ユーザーの情報はどこにあるか | プロセスごとの `struct cred` |
| 権限の判定に使うのはどの UID か | 実効 UID（ファイルはファイルシステム UID。普通は同じ値） |
| 4つが分かれるのはいつか | set-user-ID プログラムの実行時と、プログラムが自分で切り替えたとき |
| 見る方法 | `/proc/<PID>/status`、`ps -o ruid,euid,suid,fsuid`、`getresuid()` |
