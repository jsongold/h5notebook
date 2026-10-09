# 4. fork / execve での引き継ぎ

## 4.1 新しいプロセスの作り方

Linux で別のプログラムを動かすには、次の2つのシステムコールを組み合わせる。

| システムコール | すること | 資格情報（cred） |
|---|---|---|
| `fork()`（実体は `clone`） | 自分のコピーの子プロセスを作る | 親の cred をそのままコピーする |
| `execve()` | 自分のプロセスの中身を別のプログラムに置き換える | 原則そのまま。set-user-ID ビットがあるときだけ変わる |

```
親 (uid 1000) --fork--> 子 (uid 1000, 親のコピー) --execve("/usr/bin/id")--> 子 = id (uid 1000)
```

シェルでコマンドを打つと、シェルが fork し、子が execve する。alice のシェルから起動したプログラムが alice として動くのはこのためである。誰かが「このプロセスは alice」と決め直しているわけではない。

カーネルでは、`fork` の中の `copy_creds()`（`kernel/cred.c`）が `prepare_creds()` を呼び、親の cred の複製を子に付ける。

## 4.2 確かめるためのプログラム

自分の3つの UID（実、実効、保存）を表示するプログラム `showid` と、fork してから引数のプログラムを execve する `forkexec` を用意する。

```c
/* showid.c */
#define _GNU_SOURCE
#include <stdio.h>
#include <unistd.h>

int main(void) {
    uid_t r, e, s;
    gid_t gr, ge, gs;
    gid_t groups[32];
    getresuid(&r, &e, &s);
    getresgid(&gr, &ge, &gs);
    int n = getgroups(32, groups);
    printf("showid pid=%d ppid=%d uid(r,e,s)=(%d,%d,%d) gid(r,e,s)=(%d,%d,%d) groups=",
           getpid(), getppid(), r, e, s, gr, ge, gs);
    for (int i = 0; i < n; i++) printf("%s%d", i ? "," : "", groups[i]);
    printf("\n");
    return 0;
}
```

```c
/* forkexec.c */
#define _GNU_SOURCE
#include <stdio.h>
#include <unistd.h>
#include <sys/wait.h>

static void show(const char *who) {
    uid_t r, e, s;
    getresuid(&r, &e, &s);
    printf("%-6s pid=%d uid(r,e,s)=(%d,%d,%d)\n", who, getpid(), r, e, s);
    fflush(stdout);
}

int main(int argc, char *argv[]) {
    show("parent");
    pid_t pid = fork();
    if (pid == 0) {
        show("child");
        char *args[] = { argv[1], NULL };
        execv(argv[1], args);      /* 成功すれば戻らない */
        perror("execv");
        return 1;
    }
    waitpid(pid, NULL, 0);
    return 0;
}
```

```
# gcc -Wall -o /usr/local/bin/showid showid.c
# gcc -Wall -o /usr/local/bin/forkexec forkexec.c
```

## 4.3 fork と execve をまたいでも変わらない

```
alice$ forkexec /usr/local/bin/showid
parent pid=4713 uid(r,e,s)=(1000,1000,1000)
child  pid=4714 uid(r,e,s)=(1000,1000,1000)
showid pid=4714 ppid=4713 uid(r,e,s)=(1000,1000,1000) gid(r,e,s)=(1000,1000,1000) groups=1000,1003
```

- `child` と `showid` の PID は同じ 4714 である。execve はプロセスを作らず、同じプロセスの中身を入れ替えるだけだからである。
- 3つの UID も、GID も、補助グループも、親から最後まで変わっていない。

システムコールの流れを strace で見る。`-f` で子プロセスも追い、`-o` でプログラムの出力と混ざらないようにファイルに書く。

```
alice$ strace -f -o /tmp/fe.trace -e trace=clone,execve,getresuid,wait4 forkexec /usr/local/bin/showid
（プログラムの出力は省略）
alice$ cat /tmp/fe.trace
4746  execve("/usr/local/bin/forkexec", ["forkexec", "/usr/local/bin/showid"], 0xffffc27b63f0 /* 10 vars */) = 0
4746  getresuid([1000], [1000], [1000]) = 0
4746  clone(child_stack=NULL, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD, child_tidptr=0xffff9516b050) = 4747
4746  wait4(4747,  <unfinished ...>
4747  getresuid([1000], [1000], [1000]) = 0
4747  execve("/usr/local/bin/showid", ["/usr/local/bin/showid"], 0xfffff0546130 /* 10 vars */) = 0
4747  getresuid([1000], [1000], [1000]) = 0
4747  +++ exited with 0 +++
4746  <... wait4 resumed>NULL, 0, NULL) = 4747
4746  --- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=4747, si_uid=1000, si_status=0, si_utime=0, si_stime=0} ---
4746  +++ exited with 0 +++
```

- 行頭の数字は PID である。libc の `fork()` は、実際には `clone` システムコールになっている。
- 子（4747）は `clone` の直後から、何もしなくても `getresuid` で 1000 を返す。親の cred のコピーを持って生まれたからである。
- 子が終わると、親に `SIGCHLD` が届く。`si_uid=1000` は、子の実 UID である。

## 4.4 set-user-ID ビット：execve で実効 UID が変わる

`man 2 execve` には次のように書かれている。

- 実行するファイルに set-user-ID ビットがあれば、呼び出したプロセスの実効 UID を、そのファイルの所有者の UID に変える（set-group-ID ビットなら実効 GID をファイルのグループに）。
- そのあと、実効 UID を保存 UID にコピーする。
- 実 UID、実 GID、補助グループは execve では変わらない。

`showid` をコピーして root の所有にし、set-user-ID ビット（`chmod 4755` の `4`）を付ける。`ls -l` では所有者の x の位置が `s` になる。

```
# cp /usr/local/bin/showid /usr/local/bin/showid-suid
# chmod 4755 /usr/local/bin/showid-suid
# ls -l /usr/local/bin/showid /usr/local/bin/showid-suid
-rwxr-xr-x 1 root root 70760 Oct  4 07:43 /usr/local/bin/showid
-rwsr-xr-x 1 root root 70760 Oct  4 07:43 /usr/local/bin/showid-suid
```

```
alice$ forkexec /usr/local/bin/showid-suid
parent pid=4716 uid(r,e,s)=(1000,1000,1000)
child  pid=4717 uid(r,e,s)=(1000,1000,1000)
showid pid=4717 ppid=4716 uid(r,e,s)=(1000,0,0) gid(r,e,s)=(1000,1000,1000) groups=1000,1003
```

同じ PID 4717 のまま、execve の前後で UID が変わった。

| | 実 UID | 実効 UID | 保存 UID |
|---|---|---|---|
| execve の前（child） | 1000 | 1000 | 1000 |
| execve の後（showid） | 1000（そのまま） | 0（ファイルの所有者） | 0（実効 UID のコピー） |

[2章](02-process-credentials.md)で見た `passwd` の `Uid: 1000 0 0 0` は、これと同じ仕組みでできたものである。

```
# ls -l /usr/bin/passwd
-rwsr-xr-x 1 root root 72048 Dec 14  2025 /usr/bin/passwd
```

## 4.5 set-user-ID ビットが効かない場合

`man 2 execve` は、次のときに set-user-ID / set-group-ID ビットを無視すると書いている。

- プロセスに `no_new_privs` 属性が付いているとき
- ファイルシステムが `nosuid` でマウントされているとき
- プロセスが ptrace されているとき

また、スクリプト（`#!` で始まるファイル）の set-user-ID ビットも無視される（同じ man ページの Interpreter scripts の節）。

スクリプトで試す。

```
# printf "#!/bin/sh\nid\n" > /usr/local/bin/id-suid.sh
# chmod 4755 /usr/local/bin/id-suid.sh
# ls -l /usr/local/bin/id-suid.sh
-rwsr-xr-x 1 root root    13 Oct  4 07:43 /usr/local/bin/id-suid.sh
alice$ /usr/local/bin/id-suid.sh
uid=1000(alice) gid=1000(alice) groups=1000(alice),1003(dev)
```

ptrace されている場合を試す。alice が自分で strace を付けて `showid-suid` を実行する。

```
alice$ strace -o /dev/null /usr/local/bin/showid-suid
showid pid=4724 ppid=4721 uid(r,e,s)=(1000,1000,1000) gid(r,e,s)=(1000,1000,1000) groups=1000,1003
```

実効 UID が 0 にならない。もし strace の下で root になれたら、alice は root のプロセスのメモリを書き換えられてしまう。それを防ぐための規則である。[6章](06-sudo-su.md)で sudo を strace するときは、root で strace を動かし、`-u alice` で alice として sudo を起動する（`man strace` によると、`-u` は root で動かしたときに set-user-ID のプログラムを正しく実行させるためのオプションである）。

## 4.6 まとめ

| 操作 | 実 UID | 実効 UID | 保存 UID | 補助グループ |
|---|---|---|---|---|
| fork | 親と同じ | 親と同じ | 親と同じ | 親と同じ |
| execve（普通のファイル） | 変わらない | 変わらない | 実効 UID のコピー（＝変わらない） | 変わらない |
| execve（set-user-ID ビット付き） | 変わらない | ファイルの所有者 | ファイルの所有者 | 変わらない |
| execve（ptrace 中、nosuid、スクリプト） | 変わらない | 変わらない | 変わらない | 変わらない |
