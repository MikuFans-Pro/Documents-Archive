# 你知道吗？你的每一次 `sudo`，系统都悄悄记小本本上了

不是比喻，是真的记了。

事情是这样的——我前几天想在服务器上查一个操作记录，就想确认某个时间点我到底有没有不小心把 `nginx` 给停了。记性不太靠谱，但 Linux 就很靠谱。

## 你敲的每一条 sudo，都躺在日志里

Linux 默认会通过 `journald`（systemd 自带的日志系统）记录所有 `sudo` 调用。查的话也简单，一行：

```bash
journalctl _COMM=sudo
```

你会看到类似这样的东西：

```
May 29 16:23:41 myserver sudo[13892]: metalab : TTY=pts/0 ; PWD=/var/www ; USER=root ; COMMAND=/usr/bin/systemctl stop nginx
```

翻译成人话：

```
May 29 16:23:41    | 时间
metalab            | 是谁在搞事（你自己）
TTY=pts/0          | 从哪个终端过来的
PWD=/var/www       | 敲命令的时候你在哪个目录
COMMAND=...        | 你到底干了什么
```

**人证物证俱在。**

## 查得更细一点

只看某个用户的 sudo 操作（把 `metalab` 换成你自己的用户名）：

```bash
journalctl _COMM=sudo | grep metalab
```

看最近十次：

```bash
journalctl _COMM=sudo -n 10
```

想看某个时间段的——比如找 5 月 28 号那一波折腾：

```bash
journalctl _COMM=sudo --since "2026-05-28" --until "2026-05-29"
```

想看实时的（别人正在 sudo 的时候你盯着）：

```bash
journalctl _COMM=sudo -f
```

>! 视奸.sh !<

## 注意一下权限

能不能直接跑，取决于你当前用户有没有读 journal 的权限——通常单机/桌面用户默认有，但多用户服务器上可能只有 root 和有 `systemd-journal` 或 `adm` 组的用户能看。看不了就老老实实 `sudo journalctl ...`。

>! 用 sudo 查 sudo 日志，套娃了属于是。 !<

不同发行版的差异：Ubuntu/Debian 默认记录 sudo 日志到 journald，RHEL/CentOS 系可能走 `/var/log/secure`。如果不确定，先 `journalctl _COMM=sudo` 试一下，没东西再去翻 `/var/log/secure`：

```bash
grep sudo /var/log/secure
```

## 除了查自己，还有别的用

### 多服务器运维

如果你管着多台服务器，偶尔不记得自己切到什么机器上改过什么东西，`journalctl _COMM=sudo` 就是你的救星。不用翻 `.bash_history`（那玩意儿可能被覆盖，也可能不记录时间），`journald` 每一条都带时间戳。

>! 毕竟很多操作都需要sudo，比如 vim /etc/nginx/sites-available/default !<

### 安全审计

还有一个比较阴间但确实存在的应用场景：**安全审计**。如果有人溜进你的服务器敲了一堆 `sudo rm -rf`，日志其实都记下来了——但别高兴太早，能删文件的入侵者大概率也拿到了 root，顺手 `journalctl --vacuum-time=1s` 就能把现场清干净。

所以这东西更像一层**心理门槛**：不是每个入侵者都会第一时间清日志，脚本小子可能只想着搞破坏，没想过擦屁股。真碰上了高手，你看到的大概只剩一条清日志的命令，前面干了啥就随缘了。

>! 总结：能防菜鸟，防不了大佬。但话说回来，安全本来就是一层层垒的，少一层不如多一层。 !<

### 排查谁重启了服务

某个服务突然挂了，`journalctl _COMM=sudo | grep systemctl` 看看是不是有人手贱关掉了啥。

```bash
journalctl _COMM=sudo | grep --color=auto -E "restart|stop|reboot"
```

## 进阶：把 sudo 日志单独拎出来用

如果你觉得 `journalctl` 查起来不方便，可以写个 alias 少打几个字：

```bash
alias sudolog='journalctl _COMM=sudo -n 20 --no-pager'
```

加到 `~/.bashrc` 或 `~/.zshrc` 里，以后 `sudolog` 四个字母就能看最近 20 条 sudo 记录。

想更高级的，可以把 sudo 日志导出分析：

```bash
journalctl _COMM=sudo -o json --since "2026-05-01" | jq '.MESSAGE'
```

输出 JSON 格式，拿 `jq` 提取 `MESSAGE` 字段，适合做批量分析。

---

## 总结（也没啥好总结的）

就是一个小知识：你每次 `sudo`，系统都帮你记着。记性不好？让 Linux 帮你记。

---
*首发于 [MetaLab](https://lab.metazone.cc/d/15)*