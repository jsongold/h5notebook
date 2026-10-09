# 5. 権限を切り替えるシステムコール

## 5.1 4つのシステムコールと規則

プロセスが自分の cred を書き換えるシステムコールは次のとおりである。GID 用には `setgid` / `setegid` / `setresgid` がある。規則はコンテナ内の man ページ（`man 2 setuid` ほか）による。

| システムコール | 変えるもの | 特権のないプロセス | 特権のあるプロセス（`CAP_SETUID`） |
|---|---|---|---|
| `setresuid(r, e, s)` | 実・実効・保存を個別に（-1 は変えない） | 3つそれぞれ、今の実・実効・保存のどれかの値にだけ変えられる | 任意の値にできる |
| `seteuid(e)` | 実効だけ | 今の実・実効・保存のどれかの値にだけ変えられる | 任意の値にできる |
| `setuid(u)` | 特権なしなら実効だけ、特権ありなら3つとも | 今の実か保存の値にだけ変えられる | 3つとも `u` になる。もう戻れない |
| `setgroups(n, list)` | 補助グループ | できない（EPERM） | 任意のリストにできる（`CAP_SETGID` が必要） |

特権のないプロセスの規則を一言でいうと、「今持っている3つの値の間で入れ替えることはできるが、持っていない値は作れない」である。

`setresuid` は、どの UID を変えても、ファイルシステム UID を新しい実効 UID に合わせる。

カーネルの実装（`kernel/sys.c` の `__sys_setresuid()`）を簡略化すると次のようになる。

```c
old = current_cred();
/* 新しい値が、今の uid / euid / suid のどれでもないものを含むか */
if ((ruid_new || euid_new || suid_new) && !ns_capable_setid(old->user_ns, CAP_SETUID))
	return -EPERM;
new = prepare_creds();          /* 今の cred のコピーを作る */
new->uid = ruid; new->euid = euid; new->suid = suid;   /* -1 の欄は変えない */
new->fsuid = new->euid;
return commit_creds(new);       /* プロセスの cred を新しいものに差し替える */
```

判定に使うのは「root かどうか」ではなく `CAP_SETUID` を持つかどうかである。

## 5.2 一時的に降りる、完全に降りる

set-user-ID root のプログラムは、起動した瞬間から root として動く（[4章](04-fork-execve.md)）。必要なときだけ root の力を使い、それ以外では起動したユーザーとして動くのが安全である。そのための2つの降り方を、次のプログラムで確かめる。

```c
/* dropuid.c */
#define _GNU_SOURCE
#include <stdio.h>
#include <string.h>
#include <errno.h>
#include <unistd.h>

static void show(const char *step) {
    uid_t r, e, s;
    getresuid(&r, &e, &s);
    printf("%-24s uid(r,e,s)=(%d,%d,%d)\n", step, r, e, s);
}

static void try(const char *step, int rc) {
    if (rc == -1) printf("%-24s -> %s\n", step, strerror(errno));
    else show(step);
}

int main(void) {
    uid_t me = getuid();               /* 実行したユーザー（実 UID） */
    show("start");
    try("seteuid(me)", seteuid(me));          /* 一時的に降りる */
    try("seteuid(0)", seteuid(0));            /* 元に戻る */
    try("setresuid(me,me,me)", setresuid(me, me, me)); /* 完全に降りる */
    try("seteuid(0)", seteuid(0));            /* 戻ろうとする */
    return 0;
}
```

```
# gcc -Wall -o /usr/local/bin/dropuid dropuid.c
# cp /usr/local/bin/dropuid /usr/local/bin/dropuid-suid
# chmod 4755 /usr/local/bin/dropuid-suid
```

set-user-ID root にしたものを alice が実行する。

```
alice$ dropuid-suid
start                    uid(r,e,s)=(1000,0,0)
seteuid(me)              uid(r,e,s)=(1000,1000,0)
seteuid(0)               uid(r,e,s)=(1000,0,0)
setresuid(me,me,me)      uid(r,e,s)=(1000,1000,1000)
seteuid(0)               -> Operation not permitted
```

| 行 | 何が起きたか |
|---|---|
| start | execve で実効と保存が 0 になっている |
| seteuid(me) | 実効だけ 1000 に下げた。保存に 0 が残っている（一時的に降りた） |
| seteuid(0) | 0 は保存 UID の値なので、特権がなくても戻せる |
| setresuid(me,me,me) | 3つとも 1000 にした。0 はどこにも残っていない（完全に降りた） |
| seteuid(0) | 0 は今の3つのどれでもなく、`CAP_SETUID` もないので EPERM |

保存 UID は、この「一時的に降りて戻る」ための控えである。

比べるために、set-user-ID ビットのない `dropuid` を alice が実行する。

```
alice$ dropuid
start                    uid(r,e,s)=(1000,1000,1000)
seteuid(me)              uid(r,e,s)=(1000,1000,1000)
seteuid(0)               -> Operation not permitted
setresuid(me,me,me)      uid(r,e,s)=(1000,1000,1000)
seteuid(0)               -> Operation not permitted
```

最初から 0 を持っていないので、一度も root になれない。

## 5.3 root から別のユーザーになる順番

root のプロセスが別のユーザー（ここでは bob、UID 1001、GID 1001）になるには、次の順番で呼ぶ。

```
setgroups(補助グループ) → setresgid(gid, gid, gid) → setresuid(uid, uid, uid)
```

UID を最後にするのは、`setresuid` で UID 0 を手放した時点でケーパビリティが消え、`setgroups` や `setresgid` ができなくなるからである（5.4）。

各段階の後で `/proc/self/status` を表示するプログラムで確かめる。

```c
/* becomeuser.c */
#define _GNU_SOURCE
#include <stdio.h>
#include <string.h>
#include <errno.h>
#include <unistd.h>
#include <grp.h>

/* /proc/self/status の Uid / Gid / Groups / CapEff 行だけを出す */
static void status(const char *step) {
    char line[256];
    FILE *f = fopen("/proc/self/status", "r");
    printf("== %s\n", step);
    while (fgets(line, sizeof line, f))
        if (!strncmp(line, "Uid:", 4) || !strncmp(line, "Gid:", 4) ||
            !strncmp(line, "Groups:", 7) || !strncmp(line, "CapEff:", 7))
            printf("   %s", line);
    fclose(f);
}

int main(void) {
    gid_t groups[] = { 1003 };          /* dev */
    setvbuf(stdout, NULL, _IONBF, 0);   /* perror と出力順をそろえる */
    status("start");
    if (setgroups(1, groups) == -1) perror("setgroups");
    status("setgroups([1003])");
    if (setresgid(1001, 1001, 1001) == -1) perror("setresgid");
    status("setresgid(1001,1001,1001)");
    if (setresuid(1001, 1001, 1001) == -1) perror("setresuid");
    status("setresuid(1001,1001,1001)");
    if (setresuid(0, 0, 0) == -1) perror("setresuid(0,0,0)");
    return 0;
}
```

```
# gcc -Wall -o /usr/local/bin/becomeuser becomeuser.c
# becomeuser
== start
   Uid:	0	0	0	0
   Gid:	0	0	0	0
   Groups:	0 
   CapEff:	00000000a80c25fb
== setgroups([1003])
   Uid:	0	0	0	0
   Gid:	0	0	0	0
   Groups:	1003 
   CapEff:	00000000a80c25fb
== setresgid(1001,1001,1001)
   Uid:	0	0	0	0
   Gid:	1001	1001	1001	1001
   Groups:	1003 
   CapEff:	00000000a80c25fb
== setresuid(1001,1001,1001)
   Uid:	1001	1001	1001	1001
   Gid:	1001	1001	1001	1001
   Groups:	1003 
   CapEff:	0000000000000000
setresuid(0,0,0): Operation not permitted
```

- 1行ずつ、cred の該当する欄だけが変わっていく。
- `setresuid` で UID が 0 でなくなった瞬間に、`CapEff` が 0 になった。
- そのため、最後の `setresuid(0,0,0)` は失敗する。もう戻れない。

同じプログラムを alice が実行すると、すべて EPERM になり、何も変わらない。

```
alice$ becomeuser
== start
   Uid:	1000	1000	1000	1000
   Gid:	1000	1000	1000	1000
   Groups:	1000 1003 
   CapEff:	0000000000000000
setgroups: Operation not permitted
== setgroups([1003])
   Uid:	1000	1000	1000	1000
   Gid:	1000	1000	1000	1000
   Groups:	1000 1003 
   CapEff:	0000000000000000
setresgid: Operation not permitted
== setresgid(1001,1001,1001)
   Uid:	1000	1000	1000	1000
   Gid:	1000	1000	1000	1000
   Groups:	1000 1003 
   CapEff:	0000000000000000
setresuid: Operation not permitted
== setresuid(1001,1001,1001)
   Uid:	1000	1000	1000	1000
   Gid:	1000	1000	1000	1000
   Groups:	1000 1003 
   CapEff:	0000000000000000
setresuid(0,0,0): Operation not permitted
```

1001 は alice の3つの UID のどれでもないので、`setresuid(1001,...)` は許されない。他人になれないことが、カーネルの規則として表れている。

## 5.4 ケーパビリティとの関係

Linux では、root の力はケーパビリティという権限に分かれている（この環境では `/proc/sys/kernel/cap_last_cap` が 40 で、0〜40 の41個）（[3章](03-file-permission-check.md)）。プロセスは複数の集合を持ち、主に次の2つを見る。

| 集合 | /proc の行 | 意味 |
|---|---|---|
| permitted | `CapPrm` | 使ってよいケーパビリティの上限 |
| effective | `CapEff` | 今、実際に判定に使われるケーパビリティ |

`man 7 capabilities` の「Effect of user ID changes on capabilities」の節によると、UID を変えるとカーネルがケーパビリティを自動で変える。

- 実・実効・保存のどれかが 0 だったのが、すべて 0 以外になったとき：permitted と effective をすべて消す。
- 実効 UID が 0 から 0 以外になったとき：effective を消す。
- 実効 UID が 0 以外から 0 になったとき：permitted を effective にコピーする。

root で次の Python を実行して確かめる。

```python
# capstep.py
import os

def show(step):
    lines = open("/proc/self/status").read().splitlines()
    caps = [l for l in lines if l.startswith(("CapPrm", "CapEff"))]
    print(f"{step:<26} uid={os.getresuid()}  " + "  ".join(c.replace("\t", " ") for c in caps))

show("start")
os.seteuid(1000); show("seteuid(1000)")
os.seteuid(0);    show("seteuid(0)")
os.setresuid(1000, 1000, 1000); show("setresuid(1000,1000,1000)")
```

```
# python3 capstep.py
start                      uid=(0, 0, 0)  CapPrm: 00000000a80c25fb  CapEff: 00000000a80c25fb
seteuid(1000)              uid=(0, 1000, 0)  CapPrm: 00000000a80c25fb  CapEff: 0000000000000000
seteuid(0)                 uid=(0, 0, 0)  CapPrm: 00000000a80c25fb  CapEff: 00000000a80c25fb
setresuid(1000,1000,1000)  uid=(1000, 1000, 1000)  CapPrm: 0000000000000000  CapEff: 0000000000000000
```

一時的に降りたとき（`seteuid(1000)`）は permitted が残っているので、`seteuid(0)` で effective が戻る。完全に降りたとき（`setresuid(1000,1000,1000)`）は permitted ごと消える。5.2 の「戻れる・戻れない」は、ケーパビリティの側から見るとこうなっている。

ケーパビリティは `getpcaps` と `capsh --print` でも見られる。`=ep` は effective と permitted の両方にあることを表し、`=` だけなら何もない。

```
# getpcaps 1
1: cap_chown,cap_dac_override,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_net_raw,cap_sys_chroot,cap_sys_ptrace,cap_mknod,cap_audit_write,cap_setfcap=ep
alice$ capsh --print | grep -E "^(Current|uid)"
Current: =
Current IAB: !cap_dac_read_search,!cap_linux_immutable,!cap_net_broadcast,!cap_net_admin,!cap_ipc_lock,!cap_ipc_owner,!cap_sys_module,!cap_sys_rawio,!cap_sys_pacct,!cap_sys_admin,!cap_sys_boot,!cap_sys_nice,!cap_sys_resource,!cap_sys_time,!cap_sys_tty_config,!cap_lease,!cap_audit_control,!cap_mac_override,!cap_mac_admin,!cap_syslog,!cap_wake_alarm,!cap_block_suspend,!cap_audit_read,!cap_perfmon,!cap_bpf,!cap_checkpoint_restore
uid=1000(alice) euid=1000(alice)
```

alice が set-user-ID root の `passwd` を実行している最中のプロセスを見ると、実 UID は alice のままでも、root と同じケーパビリティを持っている。

```
# grep -E "^(Uid|CapPrm|CapEff)" /proc/6004/status
Uid:	1000	0	0	0
CapPrm:	00000000a80c25fb
CapEff:	00000000a80c25fb
# getpcaps 6004
6004: cap_chown,cap_dac_override,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_net_raw,cap_sys_chroot,cap_sys_ptrace,cap_mknod,cap_audit_write,cap_setfcap=ep
```

（6004 はそのときの `passwd` の PID。Docker のコンテナなので、ケーパビリティはコンテナの root が持つ範囲に限られている。）

## 5.5 まとめ

| 問い | 答え |
|---|---|
| 特権がないと何ができるか | 今持っている実・実効・保存の値の間で入れ替えることだけ |
| 一時的に降りるには | `seteuid(実 UID)`。保存 UID に元の値が残る |
| 完全に降りるには | `setresuid(u, u, u)`。元の値がどこにも残らない |
| root から別ユーザーになる順番 | setgroups → setresgid → setresuid（UID が最後） |
| 特権の実体 | `CAP_SETUID` / `CAP_SETGID` などのケーパビリティ。UID の変化に合わせてカーネルが消したり戻したりする |
