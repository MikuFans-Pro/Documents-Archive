# Flarum之八--20坑复盘与国内部署最小踩坑路径（最终章？）

从零到七，八篇文章，前后两天——两天踩了二十多个坑。回头一看，比预期多得多。

这是终结篇。不介绍新工具、不解决新问题，只做三件事：
1. **画一张坑地图**，把八篇文章里踩过的 20 个坑按类型和严重程度列出来
2. **说说哪些是可以避免的**，哪些是系统性问题
3. **给一条最小化踩坑的路线**，如果你现在要从零在国内部署 Flarum

---

## 一、旅程回顾

八篇文章，按时间线排列：

```
之零    插件开发笔记                       一个审核插件的失败经验，发现 Flarum 渲染管线耦合太深
之一    别在后台卸载扩展                   Composer Web 卸载导致 vendor/ 清空，全站 500
之二    讨论链接改成纯 ID                  /d/3-拼音-slug → /d/3，SQL 触发器一劳永逸
之三    jsdelivr 在国内挂了               emoji + 代码高亮全部依赖 jsdelivr，逐文件替换 CDN
之四    loader.min.js 依赖地狱            loader 内部 fallback 复活 jsdelivr，竞态条件导致高亮挂掉
之五    升级实录                          1.8 → 2.0 RC1，9 个坑的完整记录
之六    CDN 本地化再战 2.0                2.0 升级后 jsdelivr 残留清除 + assets:publish RC1 bug
之七    disable-email-notifications 适配  自定义扩展适配 2.0，只改两行
```

> 原帖链接：[之零](https://lab.metazone.cc/d/2) · [之一](https://lab.metazone.cc/d/3) · [之二](https://lab.metazone.cc/d/5) · [之三](https://lab.metazone.cc/d/7) · [之四](https://lab.metazone.cc/d/8) · [之五](https://lab.metazone.cc/d/9) · [之六](https://lab.metazone.cc/d/10) · [之七](https://lab.metazone.cc/d/11)

---

## 二、20 坑地图

### 类型 A：完全可避免（了解系统就能避开）

**A1 —— 后台卸载扩展（之一）**
```
严重度  : 🔴 全站挂
原因    : Flarum Web 面板卸载 = Composer remove，遇到 VCS 权限问题半途中断
如果有机会重来 : 永远走 CLI，不在面板里点卸载
```

**A2 —— slug_driver 只管解析不管生成（之二）**
```
严重度  : 🟡 不改也没事
原因    : 改完驱动以为完事，slug 列里还存着旧数据，新帖继续写拼音
如果有机会重来 : 驱动 + 触发器一起上
```

**A3 —— config.php 改 CDN 没用（之三）**
```
严重度  : 🔴 改完不生效，排查很久
原因    : emoji CDN 在 JS 里硬编码，PHP 配置根本不读
如果有机会重来 : 改之前先 grep 确认 URL 的位置
```

**A4 —— 符号链接叠目录（之三）**
```
严重度  : 🟡 多一层路径，404
原因    : JS 自己拼了 72x72/，符号链接不该再带一层
如果有机会重来 : 读懂 JS 路径拼接逻辑再建链
```

**A5 —— 漏了 repository.xml（之三）**
```
严重度  : 🔴 改完 5 个文件，回头又炸
原因    : s9e 的 BBCode XSL 模板藏在子目录里，第一轮没发现
如果有机会重来 : 全文 grep jsdelivr 看所有命中，一个不漏
```

### 类型 B：可以减少损失（提前验证可降低风险）

**B1 —— storage/formatter 编译缓存（之三）**
```
严重度  : 🔴 源文件改了，运行时读的还是旧缓存
原因    : s9e XSL 模板编译后缓存到 storage/formatter/，改了源文件必须清缓存
如果有机会重来 : 改完源文件后，三层 grep 验证（源文件/缓存/产物）
```

**B2 —— loader.min.js 内部 fallback（之四）**
```
严重度  : 🔴 前三层全修了，被 loader 内部复活
原因    : hljs-loader 的 fallback URL 默认指向 jsdelivr
如果有机会重来 : 不仅修引用，还要审计被加载组件内部的默认值
```

**B3 —— 竞态条件（之四）**
```
严重度  : 🔴 语言文件比 highlight.js 先到，hljs is not defined
原因    : 异步加载没有同步等待
如果有机会重来 : 本地加载场景下，所有异步链需要保序
```

**B4 —— lang-english 未启用（之五）**
```
严重度  : 🔴 翻译 key 全部暴露，页面挂掉
原因    : 中文语言包覆盖部分 key，未覆盖的 key 回退到英文，英文未启用 → 回退链断裂
如果有机会重来 : 升级后第一时间确认 lang-english 状态
```

**B5 —— 升级后缓存未重建（之五）**
```
严重度  : 🔴 权限组失效 + @ 通知丢失
原因    : migrate 之后权限缓存和通知逻辑没刷新
如果有机会重来 : 升级后严格按 migrate → cache:clear → assets:publish → 重启 PHP-FPM 顺序执行
```

**B6 —— 直接 cp vendor dist 里的 forum.js（之六）**
```
严重度  : 🔴 页面报错，扩展全部丢失
原因    : vendor/dist 里是未编译的核心源码，不含扩展
如果有机会重来 : 永远通过主题色切换触发前端重编译
```

### 类型 C：框架级问题（基本不可避免，但可以提前知道）

**C1 —— jsdelivr 硬编码（之三、之四）**
```
严重度  : 🔴 4 层引用，每层可能藏 jsdelivr
范围    : 源文件 → 编译缓存 → 前端产物 → 静态资源（loader 内部）
知道就行 : jsdelivr 在 Flarum 中是系统级依赖，国内部署必须全部本地化
```

**C2 —— emoji 双源问题（之三、之五、之六）**
```
严重度  : 🔴 flarum/emoji + s9e/emoji 各有一套 CDN URL
范围    : 1.x 和 2.0 的 emoji 源还不一样（jdecked twemoji 版本不同）
知道就行 : 每次升级后两处 emoji 源都要检查
```

**C3 —— hljs-loader 1.x → 2.0 源变更（之六）**
```
严重度  : 🔴 1.x 的修法在 2.0 不直接适用
范围    : 1.x = highlightjs/cdn-release，2.0 = s9e/hljs-loader
知道就行 : 大版本升级后 CDN 本地化要从头做
```

**C4 —— assets:publish RC1 bug（之六）**
```
严重度  : 🔴 rev-manifest 生成了，核心 JS 没复制
范围    : RC1 特有 bug
知道就行 : 命令跑完不代表文件到齐，必须手动检查 public/assets/
```

**C5 —— @twemoji/api base 字符串被 sed 误伤（之六）**
```
严重度  : 🔴 sed 全局替换编译产物，误改了版本号提取逻辑
知道就行 : 编译产物 patch 要边界精确，不能用全量 sed
```

**C6 —— XSLTProcessor 废弃（之四、之五）**
```
严重度  : 🟡 目前只是警告，但浏览器 2026-2027 会移除
范围    : s9e 的客户端 XSL 转换依赖此 API
知道就行 : 等 Flarum 上游修，自己别碰 s9e 源码
```

**C7 —— Flarum 审核机制只管正文（之零）**
```
严重度  : 🟡 标题不过审，编辑退回审核后帖子变空壳
范围    : 架构设计问题，小插件修不了
知道就行 : 考虑直接关掉编辑权限
```

**C8 —— 第三方扩展 composer 约束冲突（之五）**
```
严重度  : 🟡 升级时旧约束和 root 包冲突
范围    : 每次大版本升级都会遇到
知道就行 : 先移除 → 升级 → 适配 → 加回
```

**C9 —— mariadb 驱动在 Illuminate v8 不支持（之五）**
```
严重度  : 🟡 升级前改驱动报错
范围    : Laravel Illuminate 版本差异
知道就行 : 驱动切换等升级完成再改
```

---

## 三、如果你现在要从零部署 Flarum（最小化踩坑路线图）

基于上面 20 个坑的教训，整理出一条最小化踩坑的路线。按这个顺序做，能绕过绝大部分坑。

### 阶段一：部署前准备

```
1. 永远走 CLI，不用后台面板
    安装/卸载/更新全部通过 composer CLI
    后台面板只做内容管理

2. 部署上线前先把 jsdelivr 全部本地化
    别等上线后 emoji 裂图了再修
    具体参考之三 + 之四
```

### 阶段二：CDN 本地化（一次性做好，别再踩 4 层坑）

```
第一层：改源文件
    1. vendor/flarum/emoji/js/dist/forum.js        （客户端 emoji CDN）
    2. vendor/s9e/text-formatter/src/Bundles/Forum.php （hljs-loader + emoji）
    3. vendor/s9e/text-formatter/src/Bundles/Forum/Renderer.php （同上）
    4. vendor/s9e/text-formatter/src/Plugins/BBCodes/Configurator/repository.xml （XSL 模板）
    5. vendor/s9e/text-formatter/src/Plugins/Emoji/Configurator.php （emoji SVG）

第二层：清编译缓存
    rm -rf storage/formatter/* storage/cache/* storage/views/*

第三层：修 loader.min.js（最关键的一步）
    下载 s9e/hljs-loader、改内部 fallback 为自动推导、修复竞态条件
    参考之四完整步骤

第四层：重建前端产物 + 验证
    - 切主题色触发重编译（不要直接 cp vendor dist）
    - 跑三层 grep：
      grep -r "cdn.jsdelivr.net" vendor/ | grep -v "locale/"
      grep -r "cdn.jsdelivr.net" storage/
      grep -r "cdn.jsdelivr.net" public/assets/ | grep -v "\.map"
```

### 阶段三：部署后优化

```
1. 讨论链接改纯 ID（之二）
    slug_driver → 'id' + SQL 触发器

2. 语言包确认
    lang-english 必须启用（之五）

3. 不要顺便动 config.php 的数据库驱动
    如果用的是 MariaDB，升级 Illuminate 版本后再切 mariadb
```

### 阶段四：大版本升级（如 1.x → 2.0）

```
升级前：
    备份站点 tar.gz + 数据库 dump.sql
    列出所有已安装扩展，逐个确认兼容性
    第三方扩展从 composer.json 移除（升级完再适配加回）

升级中：
    composer update → php flarum migrate
    → 改 config.php 驱动（如果之前用的是伪装 mysql）
    → cache:clear → assets:publish → 重启 PHP-FPM

升级后（必须逐个验证）：
    语言包状态（lang-english + 中文）
    CDN 本地化（大版本升级后 CDN 源可能变了，必须重新做）
    assets:publish 产物完整性（手动检查 forum.js/admin.js 是否存在）
    权限组（随便找个版主号登录确认权限正常）
    @ 通知（@ 别人看能否收到）
    发帖/回复/删帖/点赞/标签/置顶/锁定/代码高亮/emoji
```

---

## 四、通用的"CDN 本地化四层模型"

这个模型不只是 Flarum 的事，任何用外部 CDN 的系统都可以用同样的思路做本地化：

```
第 1 层 → 源文件         vendor/s9e/、vendor/flarum/emoji/  硬编码 URL，改完进入第 2 层
第 2 层 → 编译缓存       storage/formatter/Renderer_*.php    s9e 编译缓存，源文件→缓存有延迟
第 3 层 → 前端产物       public/assets/forum.js              JS/CSS bundle，浏览器请求入口
第 4 层 → 静态资源内部   loader.min.js 内部 fallback          嵌套依赖，最容易漏的一层
```

每一层是叠乘关系，不是或的关系——**四层中只要有一层残留，CDN 引用就会从那个缺口复活**。

验证方法：改完每一层后都做全量 grep，最终四层全部清零才算完成。

---

## 五、Flarum 2.0 值得升吗？

升级成本 vs 收益：

```
升级成本：
    操作时间        : 15-20 分钟（纯升级）
    额外处理        : CDN 本地化重做（30 分钟）+ 自定义扩展适配（之七，2 行改动）
    已知风险        : assets:publish RC1 bug（需手动检查产物）
    遗留问题        : XSLTProcessor 废弃警告（1.x 也有）

升级收益：
    API 冻结         : RC1 起不再引入破坏性变更，正式版路径清晰
    PHP 兼容性       : Illuminate v8 → v13，长期安全维护
    扩展生态         : 15 个核心扩展全部无缝升级
    MariaDB 原生驱动 : 不再用 mysql 伪装
    Font Awesome 7.2 : 新图标库，视觉升级
```

**结论：值得升。** 坑主要在周边（CDN、语言包、自定义扩展适配），核心升级路径本身很顺畅。建议等正式版发布后再升也不迟，但 RC1 已经足够稳定可以提前验证。

---

## 六、写在最后

八篇文章，真正花在解决 Flarum 自身问题上的时间不多，大部分时间都在跟 **jsdelivr 网络问题**和**s9e 的渲染管线耦合**较劲。

如果有什么是"如果一开始就知道就能省掉大把时间"的事，就这三条：

1. **Flarum + 国内 = 必须本地化 jsdelivr**，不是"要不要做"，是"必须做"。不做的话 emoji 是裂图、代码块没高亮、后台管理页可能都加载不出来。
2. **CDN 本地化有四层**，少一层都白干。尤其是第四层（loader 内部 fallback），不看源码根本想不到。
3. **后台面板不可信**。安装、卸载、更新全走 CLI。

希望这 20 个坑能让后来的人少花一些时间。

---

*这个系列到此结束。如果以后又踩了新坑，可能写篇之终·续。但暂时先这样吧。*
