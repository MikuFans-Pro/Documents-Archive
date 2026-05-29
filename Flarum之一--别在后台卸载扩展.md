# Flarum 之一 - 别在 Flarum 后台卸载扩展，会炸

> 导读：本文是「Flarum部署连环坑」系列的一部分。完整的问题梳理与解决方案总结，请参阅：[Flarum之八--20坑复盘与国内部署最小踩坑路径（最终章？）](https://lab.metazone.cc/d/13)

刚被坑了一波，记录一下。

## 发生了什么

ACG 站点后台面板点了个"卸载"，目标是一个自己写的扩展。然后站点 500 了。

打开服务器一看，`vendor/` 目录几乎空了。只剩两个文件夹，`autoload.php` 没了，整个站挂了。

## 怎么回事

Flarum 后台的"卸载"本质上是在 Web 进程里跑 `composer remove`。

问题出在 VCS 包（通过 Git 仓库安装的扩展）上。Composer 卸载时会遍历 vendor 目录做清理，碰到其他扩展的 `.git` 目录因为权限问题删不掉，整个流程就崩了。

崩了之后的情况很恶心——不是"什么都没做"，而是"做了一半"：

- 部分文件的 `composer remove` 已经开始删了
- `.git` 目录删不掉，Composer 报错退出
- 这时候 `vendor/composer/` 里的 autoloader、installed.json 等文件已经没了
- 但包也没装回来

就卡在一个中间态，站直接没了。

## 怎么救回来的

```bash
# 清理残留
sudo rm -rf vendor/metazone vendor/flarum-lang

# 重建
cd /var/www/acg_metazone_cc
sudo -u www-data composer install --no-interaction
sudo -u www-data php flarum cache:clear
```

`composer install` 从 `composer.lock` 把所有包重新装了一遍，恢复了。

## 教训

**Flarum 后台面板的扩展管理不可信。**

安装、卸载、更新，全部走 CLI：

```bash
# 安装
编辑 composer.json → composer install

# 卸载
编辑 composer.json → composer update

# 更新
composer update <package>
```

Web 面板那层 GUI 本质上就是给 composer 套了个皮，连基本的错误恢复都没有，一崩就是全站。对自己好一点，别在面板里点那些按钮。
