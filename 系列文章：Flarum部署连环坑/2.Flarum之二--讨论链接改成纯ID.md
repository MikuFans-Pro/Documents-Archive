# Flarum 之二 - Flarum 讨论链接改成纯 ID，干掉中文拼音 slug

> 导读：本文是「Flarum部署连环坑」系列的一部分。完整的问题梳理与解决方案总结，请参阅：[Flarum之八--20坑复盘与国内部署最小踩坑路径（最终章？）](https://lab.metazone.cc/d/13)

Flarum 默认的讨论链接格式是 `/d/{id}-{slug}`，比如：

```
https://lab.metazone.cc/d/3-bie-zai-flarum-hou-tai-xie-zai-kuo-zhan-hui-zha
```

中文标题会被转成拼音 slug，又长又丑。而且标题一改 slug 跟着变，链接不稳定。

想改成干净的数字 ID：`/d/3`。

## 步骤

### 1. 改 slug 驱动

Flarum 用 `slug_driver` 控制链接格式。后台面板就能改。

或者直接写数据库：

```sql
UPDATE settings SET value = 'id'
WHERE `key` LIKE '%slug_driver_Flarum%Discussion%';
```

用户那边默认已经是 `id` 了，不用动：

```sql
-- 确认用户驱动也是 id
UPDATE settings SET value = 'id'
WHERE `key` LIKE '%slug_driver_Flarum%User%';
```

### 2. 清掉存量 slug

驱动改了，但已有帖子的 `discussions.slug` 列里还存着旧拼音，链接不会自己变。

```sql
UPDATE discussions SET slug = '';
```

### 3. 防止新帖再生成 slug 的坑

驱动只控制链接**解析**，不控制 **slug 生成**。发新帖或编辑标题时，Flarum 还是会往 `slug` 列写拼音。

**用 SQL 触发器一劳永逸**：

```sql
-- 插入时强制清空
CREATE TRIGGER clear_discussion_slug_insert
BEFORE INSERT ON discussions
FOR EACH ROW SET NEW.slug = '';

-- 更新时强制清空
CREATE TRIGGER clear_discussion_slug_update
BEFORE UPDATE ON discussions
FOR EACH ROW SET NEW.slug = '';
```

从现在起，管你发帖还是改标题，slug 永远为空，链接永远是 `/d/5`。

### 4. 清缓存

```bash
cd /var/www/你的flarum目录
sudo -u www-data php flarum cache:clear
```

## 总结

改驱动 + 触发器，三分钟搞定。比写插件省事多了。

另外，`id` 驱动有个很良心的行为：访问 `/d/3-随便什么内容` 会自动重定向到 `/d/3`。所以不用担心老链接炸——你分享出去带 slug 的旧链接，Flarum 自己就给你跳到纯 ID 了。
