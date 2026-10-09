# 6. sudo / su の中で起きていること

## 6.1 sudo は set-user-ID root のプログラム

sudo は特別な仕組みではなく、これまでの章の部品の組み合わせである。

```
# ls -l /usr/bin/sudo /usr/bin/su
-rwsr-xr-x 1 root root 133312 Nov 21  2024 /usr/bin/su
-rwsr-xr-x 1 root root 269312 Apr 11 12:01 /usr/bin/sudo
```

どちらも root 所有で set-user-ID ビットがある。alice が実行すると、execve の時点で実効 UID が 0 になる（[4章](04-fork-execve.md)）。そのあと sudo は次の順に進む。

```
alice のシェル (uid 1000)
  │ fork + execve("/usr/bin/sudo")      ← set-user-ID で実効 UID 0、実 UID 1000 のまま
  ▼
sudo (実 1000 / 実効 0)
  1. /etc/sudoers を読み、実 UID の alice が何を許されているか判定する
  2. PAM の処理（認証が必要ならパスワードの確認）
  3. setgroups → setgid             ← 補助グループと GID を対象ユーザーのものに
  4. fork
  ▼
子 (まだ root)
  5. setresuid(対象, 対象, 対象)      ← 完全に降りる（[5章](05-switching-uid.md)）
  6. execve(コマンド)
```

「誰が実行したか」を sudo が知る手がかりは、実 UID である。set-user-ID の execve でも実 UID は変わらないので、sudo は `getuid()` で alice を知ることができる。

## 6.2 strace で見る sudo -u nobody id

sudo のような set-user-ID のプログラムは、ptrace されていると set-user-ID ビットが無視される（[4章](04-fork-execve.md) 4.5）。そこで root で strace を動かし、`-u alice` で alice として sudo を起動する。UID や GID を変えるシステムコールと、プロセスの生成だけを追う。

```
# cd /tmp
# strace -f -o /tmp/sudo.trace -u alice -e trace=execve,setgroups,setresgid,setresuid,setgid,setuid,setregid,setreuid,clone,clone3,wait4 sudo -u nobody id
uid=65534(nobody) gid=65534(nogroup) groups=65534(nogroup)
# cat /tmp/sudo.trace
4934  execve("/usr/bin/sudo", ["sudo", "-u", "nobody", "id"], 0xfffff62def80 /* 7 vars */) = 0
4934  setresuid(0, -1, -1)              = 0
4934  setresgid(-1, 0, -1)              = 0
4934  setresuid(-1, 1, -1)              = 0
4934  setresuid(-1, 0, -1)              = 0
4934  setresgid(-1, -1, -1)             = 0
4934  setresuid(-1, 0, -1)              = 0
4934  setresuid(-1, 1, -1)              = 0
4934  setresuid(-1, 0, -1)              = 0
4934  setresgid(-1, -1, -1)             = 0
4934  setresuid(-1, 0, -1)              = 0
4934  setresuid(-1, 1, -1)              = 0
4934  setresuid(-1, 0, -1)              = 0
4934  setresgid(-1, -1, -1)             = 0
4934  setresuid(-1, 0, -1)              = 0
4934  setresgid(-1, 1000, -1)           = 0
4934  setresuid(1000, -1, -1)           = 0
4934  setresgid(-1, 65534, -1)          = 0
4934  setgroups(1, [65534])             = 0
4934  setresuid(-1, 65534, -1)          = 0
4934  setresuid(-1, 0, -1)              = 0
4934  setresgid(-1, 1000, -1)           = 0
4934  setgroups(2, [1000, 1003])        = 0
4934  setresuid(-1, 0, -1)              = 0
4934  setresgid(-1, 65534, -1)          = 0
4934  setgroups(1, [65534])             = 0
4934  setresuid(-1, 65534, -1)          = 0
4934  setresuid(-1, 0, -1)              = 0
4934  setresgid(-1, 1000, -1)           = 0
4934  setgroups(2, [1000, 1003])        = 0
4934  setresuid(-1, 0, -1)              = 0
4934  setgroups(1, [65534])             = 0
4934  setgid(65534)                     = 0
4934  clone(child_stack=NULL, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD, child_tidptr=0xffff9405a0f0) = 4935
4935  setresuid(65534, 65534, 65534)    = 0
4935  execve("/usr/bin/id", ["id"], 0xaaab199a9870 /* 12 vars */) = 0
4935  +++ exited with 0 +++
4934  --- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=4935, si_uid=65534, si_status=0, si_utime=0, si_stime=0} ---
4934  wait4(-1, [{WIFEXITED(s) && WEXITSTATUS(s) == 0}], WNOHANG|WSTOPPED|__WALL, NULL) = 4935
4934  wait4(-1, 0xffffd1730b60, WNOHANG|WSTOPPED|__WALL, NULL) = -1 ECHILD (No child processes)
4934  +++ exited with 0 +++
```

途中の多くの `setresuid(-1, X, -1)` は、実効 UID だけを一時的に切り替えては 0 に戻している（[5章](05-switching-uid.md) 5.2 の「一時的に降りる」）。何のために切り替えているかは、同時に開いたファイルを並べるとわかる。`openat` と `newfstatat` も追って、該当する部分だけを抜き出したものが次である。

```
# strace -f -o /tmp/sudo2.trace -u alice -e trace=execve,setgroups,setresgid,setresuid,setgid,clone,openat,newfstatat,faccessat sudo -u nobody id
# grep -v -E "ENOENT|\.so|locale|gconv|ld.so" /tmp/sudo2.trace
（抜粋）
4946  setresuid(0, -1, -1)              = 0
4946  setresgid(-1, 0, -1)              = 0
4946  setresuid(-1, 1, -1)              = 0
4946  openat(AT_FDCWD, "/etc/sudoers", O_RDONLY|O_NONBLOCK) = 5
4946  newfstatat(5, "", {st_mode=S_IFREG|0440, st_size=1714, ...}, AT_EMPTY_PATH) = 0
4946  setresuid(-1, 0, -1)              = 0
...
4946  setresgid(-1, 65534, -1)          = 0
4946  openat(AT_FDCWD, "/proc/sys/kernel/ngroups_max", O_RDONLY|O_CLOEXEC) = 6
4946  newfstatat(AT_FDCWD, "/etc/nsswitch.conf", {st_mode=S_IFREG|0644, st_size=494, ...}, 0) = 0
4946  newfstatat(AT_FDCWD, "/etc/nsswitch.conf", {st_mode=S_IFREG|0644, st_size=494, ...}, 0) = 0
4946  openat(AT_FDCWD, "/etc/group", O_RDONLY|O_CLOEXEC) = 6
4946  newfstatat(6, "", {st_mode=S_IFREG|0644, st_size=496, ...}, AT_EMPTY_PATH) = 0
4946  setgroups(1, [65534])             = 0
4946  setresuid(-1, 65534, -1)          = 0
4946  newfstatat(AT_FDCWD, "/usr/bin/id", {st_mode=S_IFREG|0755, st_size=68552, ...}, 0) = 0
4946  setresuid(-1, 0, -1)              = 0
...
4946  openat(AT_FDCWD, "/etc/pam.d/sudo", O_RDONLY) = 6
...
4946  openat(AT_FDCWD, "/etc/shadow", O_RDONLY|O_CLOEXEC) = 6
...
4946  setgroups(1, [65534])             = 0
4946  setgid(65534)                     = 0
...
4946  clone(child_stack=NULL, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD, child_tidptr=0xffffa174c0f0) = 4947
4947  setresuid(65534, 65534, 65534)    = 0
4947  execve("/usr/bin/id", ["id"], 0xaaab0246b870 /* 12 vars */) = 0
```

読み取れることを段階ごとに並べる。

| 段階 | システムコール | 意味 |
|---|---|---|
| 1. sudoers を読む | `setresgid(-1, 0, -1)` `setresuid(-1, 1, -1)` のあと `openat("/etc/sudoers")` | 実効 UID 1・実効 GID 0 で開いている。`/etc/sudoers` はモード 0440 なので、グループ（root）の r で読める（[3章](03-file-permission-check.md)）。`man sudoers` には、UID 0 を別の値に置き換えるリモートのファイルシステムに備えて、sudoers は通常グループの権限で開こうとする、と書かれている |
| 2. コマンドを確かめる | `setgroups(1, [65534])` `setresuid(-1, 65534, -1)` のあと `newfstatat("/usr/bin/id")` | 対象ユーザー（nobody）になりきってから、コマンドのファイルを調べている |
| 3. PAM | `openat("/etc/pam.d/sudo")`、`openat("/etc/shadow")` | PAM の設定を読み、パスワードのハッシュなどがある `/etc/shadow` を開いている。本書の環境は `NOPASSWD` なのでパスワードは聞かれないが、PAM の処理自体は行われている（どのモジュールのどの段階で開いたかは、この trace だけでは区別できない） |
| 4. グループを切り替える | `setgroups(1, [65534])` `setgid(65534)` | 親プロセスのうちに、補助グループと GID を対象ユーザーのものにする |
| 5. 子で UID を切り替える | `clone` → `setresuid(65534, 65534, 65534)` | 子プロセスで、3つの UID をすべて対象ユーザーにする（完全に降りる） |
| 6. 実行 | `execve("/usr/bin/id")` | 実行時の資格情報は nobody のもの |

[5章](05-switching-uid.md) 5.3 の「setgroups → setresgid → setresuid」と同じ順番で、UID が最後になっている。

`/etc/pam.d/sudo` の中身は次のとおりである。認証（auth）の設定は `common-auth` から読み込まれる。

```
# cat /etc/pam.d/sudo
#%PAM-1.0

# Set up user limits from /etc/security/limits.conf.
session    required   pam_limits.so

@include common-auth
@include common-account
@include common-session-noninteractive
# grep -v "^#" /etc/pam.d/common-auth | grep .
auth	[success=1 default=ignore]	pam_unix.so nullok
auth	requisite			pam_deny.so
auth	required			pam_permit.so
auth	optional			pam_cap.so
```

## 6.3 su の場合

su も set-user-ID root のプログラムで、流れはほぼ同じである。root から `su - bob -c id` を追うと次のようになる（root から実行したので認証は通る）。

```
# cd /srv/lab
# strace -f -qq -s 200 -o /tmp/su.trace -e trace=execve,setgroups,setresgid,setresuid,setgid,setuid,chdir,clone su - bob -c id
uid=1001(bob) gid=1001(bob) groups=1001(bob),1003(dev)
# cat /tmp/su.trace
205   execve("/usr/bin/su", ["su", "-", "bob", "-c", "id"], 0xfffff6402ec0 /* 6 vars */) = 0
205   setgroups(2, [1001, 1003])        = 0
205   clone(child_stack=NULL, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD, child_tidptr=0xffffae91fb10) = 206
205   chdir("/")                        = 0
206   setgid(1001)                      = 0
206   setuid(1001)                      = 0
206   chdir("/home/bob")                = 0
206   execve("/bin/bash", ["-bash", "-c", "id"], 0xaaab03f5dd70 /* 6 vars */) = 0
206   clone(child_stack=NULL, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD, child_tidptr=0xffff9ff780f0) = 207
207   execve("/usr/bin/id", ["id", "-u"], 0xaaab0d162710 /* 9 vars */) = 0
206   --- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=207, si_uid=1001, si_status=0, si_utime=0, si_stime=0} ---
206   execve("/usr/bin/id", ["id"], 0xaaab0d1655c0 /* 9 vars */) = 0
205   setresuid(-1, 1001, 0)            = 0
```

- su は `setuid(1001)` を使う。root が呼ぶと3つの UID がすべて 1001 になる（[5章](05-switching-uid.md) 5.1）。
- su が実行するのは、コマンドそのものではなく bob のシェル（`/etc/passwd` の7番目の欄）で、コマンドは `-c` で渡す。`-`（ログイン）を付けたので、`argv[0]` が `-bash` になり、`chdir("/home/bob")` している。`argv[0]` の先頭の `-` は、シェルに「ログインシェルとして動け」と伝える古くからの約束である。
- 途中の `id -u` は、bash がログイン時に読む `/etc/profile` の中で実行されたものである。

## 6.4 -u と -i の違い

`sudo -u nakano コマンド` は、コマンドを nakano として直接 execve する。`sudo -u nakano -i コマンド` は、nakano のログインシェルを起動し、そのシェルにコマンドを渡す。`man sudo` の `-i` の説明を要約すると次のようになる。

- 対象ユーザーのパスワードデータベースの項目にあるシェルを、ログインシェルとして実行する。`.profile` などのログイン用の設定ファイルが読まれる。
- コマンドを指定した場合は、シェルの `-c` オプションに、単純なコマンドとして渡す。コマンドと引数は、英数字・`_`・`-`・`$` 以外のすべての文字（空白を含む）をバックスラッシュでエスケープしてから、空白でつないだものになる。
- 実行の前に、対象ユーザーのホームディレクトリに移ろうとする。

alice が `/home/alice` から、2つの書き方で nakano の環境を表示する。nakano は `~/.local/bin` に pipenv を入れてある（`pip install --user` で入れたもの）。

```
alice$ sudo -u nakano sh -c "id -un; pwd; echo HOME=\$HOME; echo PATH=\$PATH; echo SHELL=\$SHELL; command -v pipenv || echo pipenv: not found"
nakano
/home/alice
HOME=/home/nakano
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
SHELL=/bin/bash
pipenv: not found
alice$ sudo -u nakano -i sh -c "id -un; pwd; echo HOME=\$HOME; echo PATH=\$PATH; echo SHELL=\$SHELL; command -v pipenv || echo pipenv: not found"
nakano
/home/nakano
HOME=/home/nakano
PATH=/home/nakano/.local/bin:/usr/local/bin:/usr/bin:/bin:/usr/local/games:/usr/games
SHELL=/bin/bash
/home/nakano/.local/bin/pipenv
```

| | `-u` だけ | `-u` と `-i` |
|---|---|---|
| UID | nakano | nakano |
| カレントディレクトリ | alice のまま（`/home/alice`） | nakano のホーム |
| HOME | nakano のもの | nakano のもの |
| PATH | sudoers の `secure_path` | nakano のログインシェルが `.profile` で作ったもの |
| nakano の `~/.local/bin` の pipenv | 見つからない | 見つかる |

- カレントディレクトリはプロセスの属性で、fork と execve で引き継がれる。`-u` だけでは誰も `chdir` しないので、alice のディレクトリのままになる。
- HOME が `-u` だけでも nakano になるのは、この環境の `/etc/sudoers` に `Defaults env_reset` があるからである。`man sudoers` の Command environment の節によると、`env_reset` では HOME、MAIL、SHELL、LOGNAME、USER を対象ユーザーにもとづいて設定する。
- PATH は `/etc/sudoers` の `Defaults secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"` になっている。

`-i` のときに実際に起きていることを strace で見る。`-s 200` で長い文字列を省略せずに出す。

```
# cd /home/alice
# strace -f -qq -s 200 -o /tmp/i.trace -e trace=execve,chdir -u alice sudo -u nakano -i sh -c "which pipenv && pipenv --version"
# grep -E "sudo|bash|chdir|/sh\"" /tmp/i.trace | head -5
5940  execve("/usr/bin/sudo", ["sudo", "-u", "nakano", "-i", "sh", "-c", "which pipenv && pipenv --version"], 0xffffc6eaaa30 /* 6 vars */) = 0
5941  chdir("/home/nakano")             = 0
5941  execve("/bin/bash", ["-bash", "--login", "-c", "sh -c which\\ pipenv\\ \\&\\&\\ pipenv\\ --version"], 0xaaaad8968d60 /* 13 vars */) = 0
5941  execve("/usr/bin/sh", ["sh", "-c", "which pipenv && pipenv --version"], 0xaaaad68bdb70 /* 15 vars */) = 0
```

1. sudo の子が `chdir("/home/nakano")` する。
2. nakano のシェル `/bin/bash` を、`argv[0]` が `-bash`、オプション `--login` のログインシェルとして起動する。`-c` に渡す文字列は、`sh -c which\ pipenv\ \&\&\ pipenv\ --version` である（strace の表示では `\` が `\\` と二重になっている）。man のとおり、空白と `&` がエスケープされている。
3. bash はこの文字列を解釈し、エスケープを外して `sh` を execve する。`sh` が受け取る3番目の引数は、元どおりの `which pipenv && pipenv --version` である。

-u だけのときは、シェルを挟まずにコマンドを直接 execve する。

```
# strace -f -qq -s 200 -o /tmp/i3.trace -e trace=execve,chdir -u alice sudo -u nakano sh -c "which pipenv && pipenv --version"
# cat /tmp/i3.trace
5958  execve("/usr/bin/sudo", ["sudo", "-u", "nakano", "sh", "-c", "which pipenv && pipenv --version"], 0xfffffb214628 /* 6 vars */) = 0
5959  execve("/usr/bin/sh", ["sh", "-c", "which pipenv && pipenv --version"], 0xaaab0ef39830 /* 12 vars */) = 0
5960  execve("/usr/bin/which", ["which", "pipenv"], 0xaaaad5932910 /* 13 vars */) = 0
5959  --- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=5960, si_uid=1002, si_status=1, si_utime=0, si_stime=0} ---
5958  --- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=5959, si_uid=1002, si_status=1, si_utime=0, si_stime=0} ---
```

`chdir` も bash もない。`which` が終了コード 1（見つからない）で終わったので、`&&` の右側は実行されていない。

## 6.5 実例：cron の持ち主が使う pipenv を確かめる

サーバーで、cron のジョブが別のユーザー（ここでは nakano）の crontab で動いているとする。ジョブが使う pipenv がどれで、どのバージョンかを、ログインしている自分（alice）から確かめたい。

pipenv が nakano の `~/.local/bin` にあり、PATH にそれを足すのが nakano の `.profile` なら、nakano のログインシェルを通さないと見つからない。そこで次のように書く。

```
alice$ sudo -u nakano -i sh -c "which pipenv && pipenv --version"
/home/nakano/.local/bin/pipenv
pipenv, version 2026.8.0
```

`-i` で nakano のログインシェルの環境（PATH、HOME、カレントディレクトリ）を作り、`sh -c "a && b"` で2つのコマンドをまとめて渡している。

ただし、cron が実行するときの環境はログインシェルとは限らない（cron は普通 `.profile` を読まない）。ここで確かめられるのは「nakano がログインしたときに見える pipenv」である。cron の行が絶対パスで pipenv を呼んでいるか、crontab の中で PATH を設定しているかは、別に確認する必要がある。

`sh -c` を付けずに書くと、どちらも意図どおりに動かない。

```
alice$ sudo -u nakano -i "which pipenv && pipenv --version"
-bash: line 1: which pipenv && pipenv --version: command not found
alice$ sudo -u nakano -i which pipenv && pipenv --version
/home/nakano/.local/bin/pipenv
bash: line 1: pipenv: command not found
```

- 1つ目は、引数が1つの文字列なので、空白も `&` も丸ごとエスケープされる。bash には `which\ pipenv\ \&\&\ pipenv\ --version` が渡され、これ全体が1つのコマンド名として探される。strace で見ると次のとおりである。

  ```
  # strace -f -qq -s 200 -o /tmp/i2.trace -e trace=execve -u alice sudo -u nakano -i "which pipenv && pipenv --version"
  # grep bash /tmp/i2.trace
  5952  execve("/bin/bash", ["-bash", "--login", "-c", "which\\ pipenv\\ \\&\\&\\ pipenv\\ --version"], 0xaaaafba4ad40 /* 13 vars */) = 0
  ```

- 2つ目は、`&&` を alice のシェルが先に解釈する。sudo が受け取るのは `which pipenv` までで、`pipenv --version` は alice として alice の PATH で実行され、見つからない。

`sh -c "..."` で包むと、`&&` を含む文字列が `sh` の1つの引数になり、sudo のエスケープは bash が外してくれるので、nakano の側の `sh` が `&&` を解釈できる。

## 6.6 プロセスの木

alice が sudo でコマンドを動かしている最中のプロセスを `ps -ef --forest` で見る。実際の端末から使うときと同じにするため、`script` で疑似端末を用意して実行した。

```
alice$ script -qc 'sudo -u nakano -i sh -c "sleep 30; echo done"' /dev/null >/dev/null 2>&1 &
# ps -ef --forest
UID          PID    PPID  C STIME TTY          TIME CMD
（root の行は省略）
alice         15       1  0 07:47 ?        00:00:00 script -qc sudo -u nakano -i sh -c "sleep 30; echo done" /dev/null
root          16      15  0 07:47 pts/0    00:00:00  \_ sudo -u nakano -i sh -c sleep 30; echo done
root          18      16  0 07:47 pts/1    00:00:00      \_ sudo -u nakano -i sh -c sleep 30; echo done
nakano        19      18  0 07:47 pts/1    00:00:00          \_ sh -c sleep 30; echo done
nakano        21      19  0 07:47 pts/1    00:00:00              \_ sleep 30
# ps -o pid,ppid,ruid,euid,ruser,user,tty,cmd --forest -C sudo,sh,sleep
    PID    PPID  RUID  EUID RUSER    USER     TT       CMD
     16      15  1000     0 alice    root     pts/0    sudo -u nakano -i sh -c sleep 30; echo done
     18      16  1000     0 alice    root     pts/1     \_ sudo -u nakano -i sh -c sleep 30; echo done
     19      18  1002  1002 nakano   nakano   pts/1         \_ sh -c sleep 30; echo done
     21      19  1002  1002 nakano   nakano   pts/1             \_ sleep 30
      1       0     0     0 root     root     ?        sleep infinity
```

- sudo のプロセスは実 UID 1000（alice）、実効 UID 0（root）である。[4章](04-fork-execve.md)の set-user-ID の execve でできた形そのものである。
- sudo が2段になっている。この環境の `/etc/sudoers` には `Defaults use_pty` があり、`man sudoers` によると、sudo が端末から実行されたときは新しい疑似端末（ここでは `pts/1`）でコマンドを動かす。2つ目の sudo はその疑似端末を管理するプロセスである。6.2 の strace では端末がなかったので（`/dev/tty` を開けずに `ENXIO`）、sudo は1段だけだった。
- nakano のログインシェルの bash が木にいない。6.4 の strace で、bash の PID のまま `sh` が execve されていたのと同じで、bash は `-c` の最後のコマンドを fork せずに execve で自分と置き換えている。
- `sleep` は `sh` が fork + execve したもので、nakano の資格情報をそのまま引き継いでいる。

## 6.7 まとめ

| 問い | 答え |
|---|---|
| sudo が root の力を得る仕組み | set-user-ID root のファイルを execve するから |
| 誰が実行したかを知る手がかり | 実 UID（set-user-ID の execve でも変わらない） |
| 対象ユーザーになる手順 | setgroups → setgid → fork → setresuid → execve |
| `-u` だけ | コマンドを直接 execve。カレントディレクトリは呼び出し元のまま、PATH は `secure_path` |
| `-u` と `-i` | 対象ユーザーのログインシェルを `--login -c` で起動。コマンドはエスケープされて1つの文字列になる |
| `sudo -u X -i sh -c "a && b"` の理由 | ログインシェルの環境で、`&&` を対象ユーザーの側で解釈させるため |
