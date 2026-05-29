# Flarum 之五 - Flarum 1.8 → 2.0 RC1 升级实录

> 导读：本文是「Flarum部署连环坑」系列的一部分。完整的问题梳理与解决方案总结，请参阅：[Flarum之八--20坑复盘与国内部署最小踩坑路径（最终章？）](https://lab.metazone.cc/d/13)

## 背景

Flarum 2.0 RC1 已经发布了（[discuss.flarum.org/d/39118](https://discuss.flarum.org/d/39118-flarum-200-rc1-released-the-last-mile-to-20)），API 已冻结，从 RC1 到 2.0.0 正式版不会再引入破坏性变更。

在往 lab.metazone.cc 上推之前，先用 acg.metazone.cc（lab 的克隆站，目前没有真实用户数据）走一遍完整流程。主要目的是验证升级路径能不能跑通、现有扩展能不能兼容、之前做的 CDN 本地化有没有被 2.0 覆盖回去。

升级方式选的是原地升级（`composer update` + `php flarum migrate`），不走重建方案，更接近 lab 未来实际会遇到的情况。

---

## 升级前快照

操作时间：2025年05月29日 9:00 ~ 12:20

```
Flarum          : 1.8.16
PHP             : 8.3.31
Composer        : 2.9.8
数据库           : MariaDB 10.11.14（driver: mysql，注意这里埋了个坑，后面会说）
站点目录         : /var/www/acg_metazone_cc
```

### 已安装扩展及升级结果

```
flarum/approval             1.8.2   → 2.0.0-rc.1
flarum/bbcode               1.8.0   → 2.0.0-rc.1
flarum/emoji                1.8.1   → 2.0.0-rc.1
flarum/extension-manager    1.0.8   → 2.0.0-rc.1
flarum/flags                1.8.2   → 2.0.0-rc.1
flarum/likes                1.8.1   → 2.0.0-rc.1
flarum/lock                 1.8.2   → 2.0.0-rc.1
flarum/markdown             1.8.1   → 2.0.0-rc.1
flarum/mentions             1.8.5   → 2.0.0-rc.1
flarum/nicknames            1.8.3   → 2.0.0-rc.1
flarum/pusher               1.8.1   → 2.0.0-rc.1
flarum/statistics           1.8.1   → 2.0.0-rc.1
flarum/sticky               1.8.2   → 2.0.0-rc.1
flarum/subscriptions        1.8.1   → 2.0.0-rc.1
flarum/suspend              1.8.6   → 2.0.0-rc.1
flarum/tags                 1.8.8   → 2.0.0-rc.1
flarum-lang/chinese-simplified   1.6.0   → 暂不兼容 2.0，先移除
flarum-lang/chinese-traditional  1.28.2  → 暂不兼容 2.0，先移除
metazone/disable-email-notifications  1.0.0  → 需适配 2.0 API，先移除
```

15 个核心扩展全部能无缝升级到 2.0.0-rc.1，这点比预想中顺利。

### 自定义扩展的兼容性预判

`metazone/disable-email-notifications` 这个是自己的扩展，升级前心里大概有数：

- `composer.json` 里的约束是 `"flarum/core": "^1.0"`，得改成 `"^1.0|^2.0"`
- 核心逻辑用的是 `Flarum\Extend\Event` + `Flarum\User\Event\Registered`，2.0 API 有没有变化不确定
- 自定义的 `RemoveEmailDriver` Extender，2.0 的 Extender 机制可能有 breaking change

后来升级时发现，因为 VCS 源里的 `composer.json` 还没更新约束，composer 检测到 `^1.0` 和 `^2.0` 冲突，干脆先移除了。等 lab 升级前再适配加回来。

---

## 升级过程

### 1. 确认备份

```bash
ls -lh backup_20260529-072619/
# acg_backup.tar.gz, acg_dump.sql, lab_backup.tar.gz, lab_dump.sql, shared_backup.tar.gz
```

备份都在，心里踏实了。

### 2. 改 config.php —— 踩了第一个坑

原本想顺手把 `config.php` 里的数据库驱动从 `mysql` 改成 `mariadb`（MariaDB 本来就应该用原生驱动）。结果改完跑 `php flarum migrate`，1.8.16 直接报 `Unsupported driver [mariadb]`。

原因很简单：Flarum 1.8.16 带的 Illuminate v8 不认识 `mariadb` 这个驱动名。只能先改回 `mysql`，等升级完成后 Illuminate 升级到 v13 再改。

```bash
# 升级完成后再执行
sed -i "s/'driver' => 'mysql'/'driver' => 'mariadb'/" config.php
sudo systemctl restart php8.3-fpm
```

### 3. 更新 composer.json

改了三处：

```json
// 1. flarum/core 版本
"flarum/core": "^2.0" // 原: "^1.8"

// 2. 自定义扩展约束
"flarum/core": "^1.0|^2.0" // 在 vendor/metazone/disable-email-notifications/composer.json

// 3. 新增
"minimum-stability": "beta"
```

`composer.lock` 里也要同步改一下 metazone/disable-email-notifications 的约束。

### 4. 跳过 1.x 更新

当前 1.8.16 已经是 1.x 最新版了，没有什么需要升级的。跑了一下 `php flarum migrate` 确认状态干净。

### 5. 禁用第三方扩展 —— 又踩坑了

```bash
php flarum extension:disable metazone-disable-email-notifications # 成功
php flarum extension:disable flarum-lang-chinese-traditional # 成功
php flarum extension:disable flarum-lang-chinese-simplified # 失败
```

简体中文语言包禁用时报了 `Cannot redeclare is_mobile()`——扩展的 `extend.php` 里有个函数重复声明的 PHP bug，应该是 2.0 兼容性问题导致的。

没办法，直接从 `composer.json` 的 require 里把三个不兼容的扩展删掉：
- `metazone/disable-email-notifications`
- `flarum-lang/chinese-simplified`
- `flarum-lang/chinese-traditional`

### 6. 执行升级

```bash
cd /var/www/acg_metazone_cc
composer update --prefer-dist --no-plugins --no-dev -a --with-all-dependencies
```

**第一次跑挂了**。composer 报自定义扩展和中文语言包的约束冲突——它们要求 `flarum/core ^1.0`，但 root 包要求 `^2.0`，两边的 VCS 源都是旧约束，composer 直接拒绝。

**解决**：从 `composer.json` require 里移除不兼容扩展（上一步已经做了），删掉 vendor 里残留的 `vendor/metazone` 和 `vendor/flarum-lang` 目录，再跑一遍。

**第二次顺利通过**：
- 22 个新安装，69 个更新，10 个移除
- flarum/core: 1.8.16 → 2.0.0-rc.1
- 15 个核心扩展全部 → 2.0.0-rc.1
- Illuminate: v8 → v13
- 新增：flarum/json-api-server 0.1.2、fortawesome/font-awesome 7.2.0、nyholm/psr7

### 7. 数据库迁移 + mariadb 驱动

这里有个小插曲——composer update 的时候 vendor 残留目录的 git 权限问题报了 `dubious ownership` 错误。因为 vendor 目录权限之前从 root 改成了 www-data，而 git 的安全策略不允许。手动删掉残留目录解决。

```bash
# 改回 mariadb 驱动（Illuminate v13 已经支持了）
sed -i "s/'driver' => 'mysql'/'driver' => 'mariadb'/" config.php

# 数据库迁移（10 个新 migration 全过）
php flarum migrate

# 修权限 + 清缓存
chown -R www-data:www-data storage
chmod -R 775 storage
php flarum cache:clear
sudo systemctl restart php8.3-fpm
```

### 8. 重建前端资源

```bash
php flarum assets:publish
# Publishing core assets...
# Publishing extension assets...
# Publishing for extension: flarum-extension-manager
```

---

## 验证

基础功能全部正常：首页能访问、帖子详情页、登录注册、发帖回复、管理员后台都没问题。

扩展方面：Tags、Markdown、BBCode、Emoji、Likes、Mentions、Sticky、Lock、Nicknames 全部正常。Mentions 的 @ 提醒后面细说。

### 语言包 —— 这次升级最大的坑

升级完之后，整个页面全是 `core.views.layout.skip_to_content`、`core.lib.meta_titles.without_page_title` 这种原始翻译 key，完全没法看。

排查了半天，发现我漏了一件关键的事：**`flarum/lang-english` 必须被显式启用。**

Flarum 的翻译机制是链式回退的：中文语言包覆盖一部分 key，没覆盖的 key 回退到英文。但如果英文也没启用，整个回退链就断了，JS 翻译直接罢工，所有 key 原样暴露。

修复步骤：
1. 启用 `flarum/lang-english`（基础回退层）
2. 安装 2.0 兼容的中文语言包：`flarum-lang/chinese-traditional` v2.0.1（有稳定 tag）、`flarum-lang/chinese-simplified` 用 `^2.0@dev`（还没打稳定 tag）
3. 启用两个中文扩展，清缓存，重发前端资源，重启 PHP-FPM

修完之后一切正常。这个坑在 lab 升级时必须记着。

### CDN/JS 回归

之前费了不少功夫做 CDN 本地化，2.0 升级后得确认没被覆盖回去。

```bash
grep -r "cdn.jsdelivr.net" vendor/s9e/ vendor/flarum/ # 15 处（源码默认值，正常）
grep -r "cdn.jsdelivr.net" storage/ # 0 处
grep -r "cdn.jsdelivr.net" public/assets/ | grep -v "\.map" # 0 处
grep -r "cdn.jsdelivr.net" /var/www/hljs/ # 1 处（.bak 文件）
```

结论：vendor 源码里 15 处 jsdelivr 引用（s9e 的 hljs-loader 和 flarum-emoji 的默认 CDN 配置），但编译产物和实际页面加载**没有任何 jsdelivr 请求**。浏览器 Console 也干净，没有网络错误，hljs-loader 确认走本地。

唯一的未知项：s9e/text-formatter 从 2.14.3 升级到了 2.19.3，之前修的 hljs-loader 竞态条件问题不知道还在不在。这个只能后续观察代码高亮的稳定性。

### 版本确认

```
Flarum core: 2.0.0-rc.1
PHP version: CLI: 8.3.31, Web: 8.3.31
PHP memory limit: CLI: -1, Web: 128M
MariaDB version: 10.11.14
Base URL: https://acg.metazone.cc
```

---

## 踩坑记录

### 坑 1：mariadb 驱动在 1.x 下不支持

升级前就想改 `mariadb`，结果 1.x 的 Illuminate v8 不认识这个驱动名。正确做法是升级完成（Illuminate v13）后再改。算是个小坑，看了报错马上就明白了。

### 坑 2：第三方扩展的 composer 约束冲突

自定义扩展和中文语言包都要求 `flarum/core ^1.0`，跟 root 包的 `^2.0` 冲突。解决方法就是先移除，升级完再加回来。语言包的 VCS 源没更新约束可以理解，下次提前在 composer.json 里改好就行。

### 坑 3：flarum-lang/chinese-simplified 禁用失败

`extend.php` 里 `is_mobile()` 函数重复声明的 bug，估计是 2.0 环境下的兼容性问题。反正要重新安装 2.0 版本的，直接移除。

### 坑 4：vendor 残留 + git dubious ownership

composer 清理旧包时遇到了 git 安全策略问题。权限从 root 改为 www-data 后，git 不认这些仓库了。手动删掉残留目录就解决了，注意点就行。

### 坑 5：语言包全部失效（关键问题）

这个前面详细说过了。**`flarum/lang-english` 是翻译回退层，不能被禁用。** 禁用它会切断整个 JS 翻译链。升级后必须确认它处于启用状态。另外简体中文还没打 2.0 稳定 tag，要用 `^2.0@dev`。

### 坑 6：XSLTProcessor 废弃警告依旧

浏览器 Console 里还在报：

```
[Deprecation] XSLTProcessor and XSLT Processing Instructions have been deprecated by all browsers...
```

位置在 `forum.js:336` 的 `init` 函数。这是 Flarum 上游的问题，1.x 就有，2.0 RC1 没修。只能等上游移除 XSLT 依赖。

### 坑 7：删除回复偶发 flarum-mentions 报错

偶尔在删回复时会触发：

```
TypeError: Cannot read properties of undefined (reading 'user')
 at addMentionedByList.js:83:28
```

`flarum-mentions` 在构建"谁提到了这条帖子"列表时，某个 mention 关联的 post 或 user 是 undefined。可能是被 @ 的帖子/用户已被删除但 mention 记录还留着，也可能是 RC1 的边界处理不完善。

影响不大，再来一次通常就正常了。推测是 RC1 的 bug，正式版可能修掉。

### 坑 8：@ 提及后对方可能收不到通知

@ 功能本身正常，但被 @ 的人似乎没收到通知。这个还需要在通知列表里进一步验证。acg 站装了 flarum/pusher，可能跟实时推送的配置有关。

### 坑 9：权限组升级后失效 —— 版主等效管理员（严重）

这个是这次测试发现的最严重的 bug。

版主（Moderator）组的权限限制**完全失效**了，实际操作权限等效管理员（Admin）。普通用户倒是正常的，没有多获得任何权限。也就是说，升级后"版主 → 管理员"这层权限梯度直接消失了。

目前还不确定具体原因。可能是 2.0 的权限机制有底层变更，1.x 的版主权限数据在 migration 后没正确映射；也可能是权限行被合并到管理员组了；或者是 2.0 的权限检查逻辑改动了。

在 lab 升级之前，这个必须修好。修复方向大概是：
1. 进后台看版主组的权限配置还在不在
2. 可能需要删掉旧版主组、重建、重新分配权限
3. 逐条核一遍版主该有的权限：编辑帖子、删除帖子、审核、锁定、置顶等

---

## 升级前后对比

```
flarum/core                    1.8.16      → 2.0.0-rc.1
s9e/text-formatter             2.14.3      → 2.19.3
Illuminate                     v8          → v13
PHP                            8.3.31      → 8.3.31
MariaDB 驱动                    mysql（伪装）→ mariadb（原生）
jsdelivr 引用（页面实际）        0           → 0
jsdelivr 引用（vendor 源码）     已手动清除   → 15 处（默认值）
核心扩展                        16          → 16（+ json-api-server）
语言包                          zh-Hans 1.6.0 → zh-Hans 2.x-dev + zh-Hant v2.0.1
自定义扩展                       disable-email 可用 → 待适配
XSLTProcessor 警告              有           → 有
```

CDN 本地化目前没有发现受到影响，这点比较放心。

---

## 总结 & lab 升级建议

整体来说，升级流程比预期顺畅。踩的坑大部分都是"改一下就好"那种，真正需要花时间处理的是权限组和自定义扩展适配。

**什么时候升**：API 已冻结，不会再有破坏性变更了，随时可以。

**升之前要准备好**：
- `metazone/disable-email-notifications` 先适配 2.0 API，Event 和 Extender 机制可能有变化
- 中文语言包：简体用 `^2.0@dev`，繁体用 `^2.0`（v2.0.1 有稳定 tag）
- 升级后一定要确认 `flarum/lang-english` 是启用状态
- 备份做好（站点 tar.gz + 数据库 dump.sql）
- `config.php` 的 `driver: mariadb` 等升级完成再改

**升级顺序**：

```
composer update → php flarum migrate → 改 config.php → cache:clear → assets:publish → 重启 PHP-FPM → 验证语言包状态
```

**升完检查**：发帖、回复、删帖、点赞、@、置顶、锁定、标签、Markdown、代码高亮、邮件通知，这些都过一遍。

**已知遗留问题**（1.8 也有的）：
- XSLTProcessor 废弃警告，等 Flarum 上游修
- ~~`favicon.ico` 404~~（实际是自己没上传图标文件，不是 bug）

**回滚**：恢复备份的 tar.gz + dump.sql，五分钟内能回去。

预估 lab 升级操作 15-20 分钟，权限组重建额外需要一些时间。等 disable-email-notifications 适配完、权限组问题在本地验证通过后，就可以往 lab 上推了。

---

### 更新一下后续（坑 9 权限 + 坑 8 通知）

**两个坑的原因一致：升级后缓存没重建。**

lab 升级时严格按顺序执行了 `cache:clear` → `assets:publish` → 重启 `php8.3-fpm`，之后一切正常：

- **坑 9 权限组**：lab 升级后 `group_permission` 表中 Mod 组 14 条权限完好无损，权限梯度正常，版主 ≠ 管理员。ACG 当时报"权限失效"是因为 `migrate` 之后 Flarum 的权限缓存还没刷新，读到空缓存就走了宽松的默认策略。后来 ACG 也清了缓存重新跑了 `assets:publish`，确认数据库权限记录本身没问题。

- **坑 8 @ 通知**：lab 实测 @ 提及后通知中心正常收到提醒。**跟 Pusher 插件毫无关联**——flarum/pusher 根本就没启用过。ACG 测试时 `cache:clear` 和前端编译没完成，通知逻辑没加载入，误以为坏了。

结论：**两个都不是 Flarum 2.0 的 bug，纯粹是缓存问题**。升级后 `migrate` 完别忘了跑 `cache:clear` + `assets:publish` + 重启 PHP-FPM 就行。
