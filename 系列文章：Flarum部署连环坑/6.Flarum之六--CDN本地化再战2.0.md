# Flarum 之六 - CDN 本地化再战——2.0 升级后 jsdelivr 残留清除

> 导读：本文是「Flarum部署连环坑」系列的一部分。完整的问题梳理与解决方案总结，请参阅：[Flarum之八--20坑复盘与国内部署最小踩坑路径（最终章？）](https://lab.metazone.cc/d/13)

## 背景

ACG 站升级到 Flarum 2.0 RC1 之后，浏览器表面上没有报 CDN 相关的错误，但拿 grep 一扫就发现编译产物和缓存里仍然残留着 jsdelivr 的引用。这些引用虽然不一定会触发请求，但留着终究是个隐患。

为了给 lab 的升级积累经验，决定把这次清除过程完整走一遍，记录下来。

## 发现了什么

升级之后对全站做了四层 grep，结果如下：

```
forum.js                             2 处  实际被浏览器加载  ⚠️
admin.js                             1 处  实际被浏览器加载  ⚠️
storage/formatter/Renderer_*.php     2 处  s9e 编译缓存
storage/locale/catalogue.*.php       3 处  翻译文本（不发起请求）
vendor/flarum/emoji/extend.php       1 处  源码默认值
vendor/flarum/emoji/locale/en.yml    1 处  翻译文本
```

重点在前两行——`forum.js` 和 `admin.js` 里的引用是浏览器真实会去请求的，必须干掉。

### 三处硬编码的 jsdelivr URL

**forum.js**（2 处）：
```
1. cdn.jsdelivr.net/gh/s9e/hljs-loader@1.0.36/loader.min.js
2. cdn.jsdelivr.net/gh/jdecked/twemoji@17.0.2/assets/
```

**admin.js**（1 处）：
```
1. cdn.jsdelivr.net/gh/jdecked/twemoji@17.0.2/assets/
```

### 跟 1.x 的差异

2.0 里虽然同样是代码高亮和 emoji 的 CDN 本地化问题，但具体引用的地址已经变了：

- **hljs-loader**：1.x 时代直接加载的是 `highlightjs/cdn-release`，2.0 换成了 s9e 自己维护的 `s9e/hljs-loader@1.0.36`，修法思路一样，但 URL 完全不同。
- **emoji**：1.x 用的 emoji 也是 jsdelivr 的，之前已经修过。2.0 换了源（`jdecked/twemoji@17.0.2`），URL 也跟着变了。

所以就算 1.x 时代做过一轮清理，升级后还是得重新搞一遍。

## 修复策略

### Emoji——最简单的一项

Flarum 2.0 的 emoji 扩展支持通过设置面板配置 CDN 地址。可以直接在数据库里设一个本地 CDN 路径，不需要改 PHP 源码：

```bash
# 方式 A：数据库直接设置
# INSERT INTO settings (key, value) VALUES ('flarum-emoji.cdn', 'https://acg.metazone.cc/twemoji/');

# 方式 B：改源码默认值（推荐同时做，防止升级后回滚覆盖）
# sed 替换 extend.php 里的 jsdelivr 为本地 CDN
```

源码里也改一份作为兜底，这样即使数据库配置丢了也不会回退到 jsdelivr。

### hljs-loader——需要改源码 + 清缓存

比较麻烦的一项，因为 URL 不是通过配置控制的，而是硬编码在 s9e 的多处源码里。

**第一步**：找到所有引用 hljs-loader 的源文件：

```bash
grep -rn "hljs-loader" vendor/s9e/ --include="*.php" | grep -v storage
```

**第二步**：将所有 jsdelivr 的 hljs-loader URL 替换为本地地址：

```bash
# vendor/s9e/ 中所有 hljs-loader 的 jsdelivr URL → 本地 CDN
sed -i 's|https://cdn.jsdelivr.net/gh/s9e/hljs-loader@[0-9.]\+/loader.min.js|https://acg.metazone.cc/hljs/loader.min.js|g' <源文件>
```

**第三步**：本地 loader 版本更新。2.0 用的是 `hljs-loader@1.0.36`，如果本地的 loader 还是旧版本，需要下载新版并重新打上 fallback URL 自动推导 + 竞态条件修复的补丁（补丁策略与帖子 7 中 1.x 版本一致）。

**第四步**：清理缓存并重建：

```bash
rm -rf storage/formatter/*
php flarum cache:clear
php flarum assets:publish
```

---

## 实际操作记录

### 1. 准备 hljs-loader@1.0.36

2.0 的 s9e 用的是 `s9e/hljs-loader@1.0.36`，而非 1.x 时代的 `highlightjs/cdn-release`，因此需要：

- 下载 1.0.36 版本的 loader
- 打上三处补丁（与帖子 7 中 1.x 版本结构相同：L flag、L=!0、L guard、fallback URL 自动推导）
- 替换服务器上已有的 loader 文件

```bash
# 下载新版 loader
curl -sL 'https://cdn.jsdelivr.net/gh/s9e/hljs-loader@1.0.36/loader.min.js' > loader-1.0.36.js

# 三处补丁（结构与 1.x 版本一致）
sed -i 's/m=!1,g/m=!1,L=!1,g/' loader-patched.js # 新增 L flag
sed -i 's/...onload callback.../...L=!0;n().../' loader-patched.js # L=!0 标记加载完成
python3 -c "..." loader-patched.js # L guard 防竞态
sed -i 's#l.hljsUrl||"https://cdn.jsdelivr.net/...#自动推导#g' loader-patched.js # fallback URL

# 上传替换服务器文件
scp loader-patched.js <user>@<server>:/tmp/
sudo cp /tmp/loader-patched.js <本地 hljs 目录>/loader.min.js
sudo chown www-data:www-data <本地 hljs 目录>/loader.min.js
```

### 2. 修改 s9e 源码（4 个文件）

2.0 的 s9e 有两个渲染器 bundle，都需要处理：

```
Forum.php                       hljs-loader URL + emoji URL
Forum/Renderer.php              hljs-loader URL（CODE 标签）+ emoji URL
BBCodes/Configurator/repository.xml  hljs-loader + highlight.js CDN URL
Emoji/Configurator.php          emoji CDN URL
```

> 以上路径前缀均为 `vendor/s9e/text-formatter/src/Bundles/` 或 `src/Plugins/`

批量替换命令：

```bash
# hljs-loader: jsdelivr → 本地 CDN
sed -i "s|https://cdn.jsdelivr.net/gh/s9e/hljs-loader@1.0.36/loader.min.js|https://acg.metazone.cc/hljs/loader.min.js|g" \
  vendor/s9e/text-formatter/src/Bundles/Forum.php \
  vendor/s9e/text-formatter/src/Bundles/Forum/Renderer.php \
  vendor/s9e/text-formatter/src/Plugins/BBCodes/Configurator/repository.xml

# highlight.js CDN → 本地 hljs 目录
sed -i "s|https://cdn.jsdelivr.net/gh/highlightjs/cdn-release@11.11.0/build/|https://acg.metazone.cc/hljs/|g" \
  vendor/s9e/text-formatter/src/Plugins/BBCodes/Configurator/repository.xml

# emoji CDN → 本地
sed -i "s|https://cdn.jsdelivr.net/gh/twitter/twemoji@latest/assets/svg/|https://acg.metazone.cc/twemoji/svg/|g" \
  vendor/s9e/text-formatter/src/Bundles/Forum.php \
  vendor/s9e/text-formatter/src/Bundles/Forum/Renderer.php \
  vendor/s9e/text-formatter/src/Plugins/Emoji/Configurator.php

# flarum-emoji 扩展默认值（注意 [version] 需转义为 \[version\]）
sed -i "s|https://cdn.jsdelivr.net/gh/jdecked/twemoji@\\[version\\]/assets/|https://acg.metazone.cc/twemoji/|g" \
  vendor/flarum/emoji/extend.php
```

> **提醒**：emoji 有双源——flarum/emoji 的 `extend.php` 和 s9e 的 `Emoji/Configurator.php`，两处都要改，漏了任何一个都会有残留。

### 3. 清缓存 + 重建 + 重启

```bash
rm -rf storage/formatter/*
php flarum cache:clear
php flarum assets:publish
sudo systemctl restart php-fpm
```

---

## 验证结果

修复完成后，重新做四层 grep：

```
public/assets: 0 处 jsdelivr                     ✅
storage/formatter: 0 处 jsdelivr                  ✅
vendor/s9e: 0 处 jsdelivr（源码已全部替换为本地 URL） ✅
vendor/flarum: 3 处 jsdelivr（locale/en.yml 翻译文本，不发起请求） ⚠️
```

- [x] public/assets 清零
- [x] storage/formatter 清零
- [x] vendor/s9e 无 jsdelivr（hljs-loader + highlight.js + emoji 全部本地化）
- [x] forum.js 中无 hljs-loader / twemoji 引用
- [x] 浏览器 Console 无 CDN 请求
- [x] 本地 hljs-loader 已更新为 1.0.36 + 补丁

### forum.js 逐项确认

```bash
$ grep -c 'hljs-loader' public/assets/forum.js
0
$ grep -c 'twemoji' public/assets/forum.js
0
$ grep -c 'jsdelivr' public/assets/forum.js
0
```

> forum.js 本身不直接包含 loader 或 twemoji 的完整 URL——这些地址由服务器端 PHP 渲染器在处理帖子内容时动态拼入 HTML，forum.js 只负责与之配合的 JS 回调逻辑。所以 grep 清零说明服务器端生成的那一端也干净了。

---

## 意外发现：`assets:publish` 不复制核心 JS（RC1 的坑）

CDN 修复完成后，执行 `cache:clear` + `assets:publish` + 重启 PHP-FPM，结果打开页面发现**全站挂了**：

```
Uncaught TypeError: app.load is not a function
 at (索引):125
```

### 排查过程

看 Nginx 错误日志：

```
forum.js → "GET /assets/forum-442d076e.js" 404
forum.css → "GET /assets/forum.css" 404
forum-zh-Hans → "GET /assets/forum-zh-Hans.js" 404
```

`rev-manifest.json` 里明明写了 `forum.js: "forum-442d076e.js"`，但 `public/assets/` 下根本没有这个文件。进一步确认：

```bash
ls -l public/assets/forum*.js          # 空的 —— RC1 的 assets:publish 没生成
ls -l vendor/flarum/core/js/dist/forum.js  # 461K —— 源码包里有
ls -l vendor/flarum/core/js/dist/admin.js  # 495K —— 源码包里有
```

**结论**：`php flarum assets:publish` 在 RC1 里存在一个 bug——`rev-manifest.json` 能正常生成，但核心 JS 文件没有被复制到 `public/assets/`。扩展的 assets（如 `flarum-extension-manager`）倒是正常发布了，唯独 Flarum 核心的 JS/CSS 丢掉了。

> 这个 bug 在 lab 升级时必须特别注意——`assets:publish` 跑完不代表 JS 文件就位了，务必手动检查 `public/assets/forum.js` 和 `admin.js` 是否存在。如果缺失，需要手动从 `vendor/flarum/core/js/dist/` 复制或通过切换主题色触发重编译。

---

## 结论

### 2.0 vs 1.x 的 CDN 本地化差异对比

```
hljs-loader 源           1.x: highlightjs/cdn-release@11.9.0
                          → 2.0: s9e/hljs-loader@1.0.36

highlight.js 加载方式     1.x: 直接加载 highlight.min.js
                          → 2.0: 通过 loader 间接加载

emoji 源                 1.x: jdecked/twemoji@15.1.0
                          → 2.0: jdecked/twemoji@17.0.2

s9e emoji 源             1.x: 无
                          → 2.0: twitter/twemoji@latest（SEO plugin）

需要修改的文件数          1.x: ~5 个
                          → 2.0: 8 个（多了 Forum.php + Forum/Renderer.php）

formatter 缓存           1.x: Renderer_*.php 1 个文件
                          → 2.0: 影响范围相同，但缓存中的 URL 不同
```

### lab 升级时的操作清单

1. **升级后立即执行** emoji + hljs CDN 本地化，否则页面实际会发出 jsdelivr 请求
2. **必须同时修改两个 Renderer**：`Bundles/Forum.php` 和 `Bundles/Forum/Renderer.php`，只修一个不够
3. **本地 loader 需同步更新**：下载 `hljs-loader@1.0.36`，重新打 fallback + 竞态补丁
4. **emoji 有双源**：flarum/emoji 的 `extend.php` + s9e 的 `Emoji/Configurator.php` 都要改
5. **sed 中 `[version]` 需转义**：写成 `\\[version\\]` 而非 `[version]`，否则会被解释为字符集
6. 修完后必须执行 `cache:clear` + `assets:publish` + 重启 PHP-FPM
7. **`assets:publish` 完成后务必检查** `public/assets/forum.js` 是否存在（RC1 有这个坑，可能只生成了 manifest 但没复制核心 JS）。如果缺失，切勿直接复制 vendor dist 里的未编译版本（见下文），应通过切换主题色触发前端重编译
8. 残留的 `locale/en.yml` 里的 jsdelivr 字面量不影响实际请求，无需处理

---

## 当前状态（2025-05-29 最终）

```
ACG 站 2.0 RC1 升级                        ✅ 完成（见帖子 8）
CDN jsdelivr 残留清除（vendor 源码）         ✅ 完成
hljs-loader 1.0.36 + patch 更新            ✅ 完成
前端重编译（forum/admin JS+CSS）            ✅ 通过主题色切换触发重编译
forum.css / forum-zh-Hans.js              ✅ 已生成（231K / 41K）
admin.css / admin-zh-Hans.js              ✅ 已生成（210K / 77K）
flarum-emoji twemoji CDN → 本地           ✅ DB 配置 + 编译产物 patch
编译产物 jsdelivr 残留                      ✅ 全部清零（除翻译文本）
权限组迁移失效分析                          ⚠️ 待处理（帖子 8 坑 9）
disable-email-notifications 适配 2.0      ⚠️ 待处理
lab 站升级                                 ⚠️ 待前置条件就绪
```

### 关键教训：不要手动复制 vendor dist 里的 JS

之前一度想用最简单的方式——直接 `cp vendor/flarum/core/js/dist/forum.js public/assets/forum.js`，但这是**错误的**：

- `vendor/dist/` 里的 `forum.js` 是**未编译**的核心源码（约 471K），不含任何已启用的扩展代码
- 正确编译后的 `forum.js` 应该包含 tags、emoji、mentions 等所有扩展的 JS 模块，体积约 700K
- 直接复制核心 JS 会导致页面报 `Store.ts "tags not registered"` 之类的错误，CSS 也会缺失

**正确的做法**：通过切换 `theme_primary_color` 触发 Flarum 的前端重编译流程：

```bash
# 临时切换主题色，触发热编译
# UPDATE settings SET value = '<临时颜色>' WHERE key = 'theme_primary_color';
php flarum cache:clear
# 访问站点触发编译
curl https://acg.metazone.cc/
# 还原主题色
# UPDATE settings SET value = '<原颜色>' WHERE key = 'theme_primary_color';
```

重编译后 `forum.js` 就包含所有扩展代码了。但还有一项额外处理——flarum-emoji 对 twemoji CDN URL（`jdecked/twemoji@17.0.2`）的引用在编译产物中仍然存在，需要通过 DB 配置 `flarum-emoji.cdn` 加 sed patch 编译产物双重修复。

---

### 更新一下后续

上面提到的 sed patch 编译产物有个副作用：全局替换会把 `@twemoji/api` 包内嵌的 `base` 字符串也改了。

具体来说，编译后的 `forum.js` 里有一段：

```javascript
const D = /([0-9]+)\.[0-9]+\.[0-9]+/g.exec(w.base)[1];
```

`w` 是 `@twemoji/api` 包，`w.base` 原值是带版本号的 jsdelivr URL（如 `...jdecked/twemoji@17.0.2/assets/`），sed 替换后变成了本地路径 `...lab.metazone.cc/twemoji/`。这导致正则匹配不到版本号、`exec` 返回 `null`、`[1]` 直接报错：

```
cdn.ts:3 Uncaught TypeError: Cannot read properties of null (reading '1')
```

**修复**：在编译产物中把版本号提取直接硬编码为 `"17"`：

```bash
python3 -c "
for site in ['lab', 'acg']:
  path = f'/var/www/{site}_metazone_cc/public/assets/forum.js'
  with open(path) as f: content = f.read()
  content = content.replace(r'/([0-9]+)\\.[0-9]+\\.[0-9]+/g.exec(w.base)[1]', '\"17\"')
  with open(path, 'w') as f: f.write(content)
"
sudo systemctl restart php8.3-fpm
```

本地 twemoji 目录本来就不依赖版本号路径（用的 `flarum-emoji.cdn` 设置里不带 `[version]`），所以写死没问题。

**教训**：以后 sed 全局替换编译产物时要更小心，`@twemoji/api` 的 `base` 字符串是用来提取版本号的，不能动。
