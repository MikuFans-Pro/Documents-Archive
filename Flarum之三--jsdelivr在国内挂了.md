# Flarum 之三 - Flarum 部署后的连环坑之：jsdelivr 在国内挂了

> 导读：本文是「Flarum部署连环坑」系列的一部分。完整的问题梳理与解决方案总结，请参阅：[Flarum之八--20坑复盘与国内部署最小踩坑路径（最终章？）](https://lab.metazone.cc/d/13)

继纯 ID、后台卸载、审核插件之后，第四个坑。

上线后发现帖子里所有 emoji 都是裂图，控制台一片红：

```
GET https://cdn.jsdelivr.net/gh/twitter/twemoji@14/assets/72x72/1f44b.png
net::ERR_CONNECTION_REFUSED
```

jsdelivr 在国内基本连不上，懂的都懂。Flarum 的 emoji 插件和 s9e 语法高亮都死在这上面：

- **emoji**：`cdn.jsdelivr.net/gh/twitter/twemoji@14/assets/...`
- **代码高亮**：`cdn.jsdelivr.net/gh/s9e/hljs-loader@1.0.34/loader.min.js`

## 第一反应：改 config.php

以为改个配置就行，往 `config.php` 塞了一行：

```php
'flarum-emoji' => array(
  'cdn' => 'https://lab.metazone.cc/twemoji/',
),
```

擦缓存，刷新，没用。

翻源码才发现 CDN 地址是**硬编码**在 `vendor/flarum/emoji/js/src/forum/cdn.js` 里的：

```javascript
export default `https://cdn.jsdelivr.net/gh/twitter/twemoji@${version}/assets/`;
```

config.php 那行根本没人读。只能直接改编译后的 JS。

## 下载资源 + 改 JS

从 GitHub 拖了 Twemoji v14.0.2 的 72x72 PNG，三千多张，放到 `/var/www/twemoji/`。

然后找到编译后的 `vendor/flarum/emoji/js/dist/forum.js`，把 jsdelivr 那段替换成自家域名：

```diff
- const y="https://cdn.jsdelivr.net/gh/twitter/twemoji@"+/([0-9]+)...+"/assets/"
+ const y="https://lab.metazone.cc/twemoji/"
```

两个站点各改各的域名。

```
lab: lab.metazone.cc/twemoji/
acg: acg.metazone.cc/twemoji/
```

## 符号链接的坑

改完 JS，`php flarum assets:publish` 重新发布，资源也放好了，刷新还是 404。

JS 里拼的路径是 `y + "72x72/" + code + ".png"`，完整 URL 长这样：

```
https://lab.metazone.cc/twemoji/72x72/1f44b.png
```

我已经把符号链接指到了 `/var/www/twemoji/72x72/`，结果 nginx 解析出来的路径变成：

```
/var/www/twemoji/72x72/72x72/1f44b.png ← 叠了
```

链接应该指向上级目录 `/var/www/twemoji/`，让 `72x72/` 走 URL 路径自然拼接。

## 最终步骤

```bash
# 1. 下载 Twemoji
# https://github.com/twitter/twemoji/releases/tag/v14.0.2
# 解压 72x72 到 /var/www/twemoji/72x72/

# 2. 建符号链接（注意别多一层 72x72）
sudo ln -sf /var/www/twemoji /var/www/你的flarum/public/twemoji

# 3. 改 vendor/flarum/emoji/js/dist/forum.js
# 搜 "jsdelivr" 替换为你的域名，注意保留后面的路径拼接逻辑

# 4. 重新发布资源
sudo -u www-data php flarum cache:clear
sudo -u www-data php flarum assets:publish
```

## 代码高亮也不行了

emoji 的问题刚解决，发现帖子里所有代码块都没高亮了。F12 一看：

```
GET https://cdn.jsdelivr.net/gh/s9e/hljs-loader@1.0.34/loader.min.js
net::ERR_CONNECTION_REFUSED
```

又是 jsdelivr。这次涉及的文件更多——Flarum 的代码高亮由 `s9e/text-formatter` 实现，CDN 引用散落在 **3 个 PHP 文件**和 **1 个编译后的 JS 里**，一共 5 处：

**hljs-loader（3 处）**

`s9e/hljs-loader` 的做法挺巧的：不在页面初始化时加载整个 highlight.js（太重），而是等浏览器空闲时按需加载。默认从 jsdelivr 拉取：

```
vendor/s9e/text-formatter/src/Bundles/Forum/Renderer.php # 2 处
vendor/s9e/text-formatter/src/Bundles/Forum.php # 1 处（XSL 模板）
```

修改思路：

```php
// 原来
src="https://cdn.jsdelivr.net/gh/s9e/hljs-loader@1.0.34/loader.min.js"

// 改成
src="https://lab.metazone.cc/hljs/loader.min.js"
data-hljs-url="https://lab.metazone.cc/hljs/"
```

`data-hljs-url` 是关键。hljs-loader 内部会从这里拼路径：

```
{data-hljs-url}highlight.min.js # 核心库
{data-hljs-url}languages/{lang}.min.js # 语言文件
```

不加这个属性的话，loader 会 fallback 到 jsdelivr。

**服务器端 emoji SVG 回退（3 处）**

同一个文件里还有 Twemoji SVG 的引用：

```php
// s9e 在服务器端渲染帖子内容时，会生成 emoji 的 SVG 标签作为 JS 加载失败时的回退
src="https://cdn.jsdelivr.net/gh/twitter/twemoji@latest/assets/svg/{@tseq}.svg"
```

直接复用本地 PNG 就行——反正服务器端回退也不在乎格式：

```php
src="{domain}/twemoji/72x72/{@tseq}.png"
```

## 下载 highlight.js

从 GitHub 拖了 `highlightjs/cdn-release` 的 v11.9.0：

```
/var/www/hljs/
├── loader.min.js # s9e/hljs-loader@1.0.34 (2.9KB)
├── highlight.min.js # highlight.js 核心 (121KB)
├── languages/ # 192 语言定义
│   ├── javascript.min.js
│   ├── python.min.js
│   └── ...
└── styles/ # 147 主题 CSS
    ├── github.min.css
    ├── monokai.min.css
    └── ...
```

两个站点共用一份文件，通过符号链接：

```bash
sudo ln -sf /var/www/hljs /var/www/lab_metazone_cc/public/hljs
sudo ln -sf /var/www/hljs /var/www/acg_metazone_cc/public/hljs
```

## 涉及的文件

以下是每个站点需要修改的 **6 个文件 + 1 层编译缓存**，删除了所有 jsdelivr 引用：

```
vendor/flarum/emoji/js/dist/forum.js                     -> 客户端 emoji CDN URL
vendor/s9e/text-formatter/src/Plugins/Emoji/Configurator.php -> 服务端 emoji SVG 回退
vendor/s9e/text-formatter/src/Bundles/Forum/Renderer.php     -> hljs-loader + emoji SVG（4处）
vendor/s9e/text-formatter/src/Bundles/Forum.php               -> hljs-loader + emoji（XSL 模板，2处）
vendor/s9e/text-formatter/src/Plugins/BBCodes/Configurator/repository.xml -> hljs-loader + highlight.js CDN（3处）
public/assets/forum.js                                        -> 编译产物，重建后自动更新
storage/formatter/Renderer_*.php                              -> s9e 编译缓存，必须清除
```

## 结论

Flarum 对国内部署有俩隐性前提：jsdelivr 能用，composer 源能连。第一个踩完了，踩了五个坑：

1. config.php 改 CDN 配置没用——JS 里硬编码的
2. 符号链接贴错层级——JS 自己拼了 `72x72/`，链接不应再带一层
3. hljs-loader 有隐藏依赖——需要把 highlight.js 核心库一起本地化
4. forum.js 不能手动生成——必须通过设置变更触发，且设置值必须合法
5. settings 表改坏值会导致全站 500——LESS 编译失败，不是寻常的报错页面

如果你也在国内部署 Flarum，建议上线前就把 jsdelivr 全部换掉，别等炸了再修。

---

**更新一下（写完这帖子的第二天）：**

又炸了。

刷新几次详情页之后突然一直转圈，控制台：

```
CommentPost.js:92 GET https://cdn.jsdelivr.net/gh/s9e/hljs-loader@1.0.34/loader.min.js
net::ERR_CONNECTION_TIMED_OUT
```

我都惊了。论坛首页和列表页没事，只有点进帖子详情页才触发。不是都改完了吗？？？

搜了一圈，`CommentPost.js` 是 Flarum 核心里处理帖子回复视图的组件（`vendor/flarum/core/js/src/forum/components/CommentPost.js`）。详情页进去后，代码块需要高亮，这里会动态创建 `<script>` 标签去加载 hljs-loader——URL 是从 s9e PHP 注入的 data 属性读的。

那问题就简单了：要么 s9e PHP 没改干净，要么 forum.js 里还有残留。

直接全局搜：

```bash
grep -rn "jsdelivr" vendor/s9e/ public/assets/
```

发现 `vendor/s9e/text-formatter/src/Plugins/BBCodes/Configurator/repository.xml` 里还藏着 3 处 jsdelivr。这是 s9e 的 `[CODE]` BBCode XSL 模板，第一次改的时候注意力全在 `Forum/Renderer.php` 和 `Forum.php` 上，完全没注意到 BBCodes 子目录下还有一个模板：

```xml
<!-- repository.xml 第 63、65、70 行 -->
<xsl:if test="'https://cdn.jsdelivr.net/gh/highlightjs/cdn-release@11.9.0/build/' != 
 '...<var name='url' description='highlight.js CDN URL'>
 https://cdn.jsdelivr.net/gh/highlightjs/cdn-release@11.9.0/build/</var>...'">
  <xsl:attribute name="data-hljs-url">...<var name='url'>
  https://cdn.jsdelivr.net/gh/highlightjs/cdn-release@11.9.0/build/</var>...</xsl:attribute>
</xsl:if>
...
<xsl:attribute name="src">
  https://cdn.jsdelivr.net/gh/s9e/hljs-loader@1.0.34/loader.min.js
</xsl:attribute>
```

3 处都要改，而且模板里还带了个 `integrity` 属性（sha384 哈希），那是对 jsdelivr 上原始文件的校验值，本地文件不匹配会导致加载被浏览器直接拦截。得一起干掉。

```bash
# 两个站点的 repository.xml 各改各的域名
# LAB → lab.metazone.cc/hljs/
# ACG → acg.metazone.cc/hljs/

sed -i \
  -e "s|https://cdn.jsdelivr.net/gh/highlightjs/cdn-release@11.9.0/build/|https://{domain}/hljs/|g" \
  -e "s|https://cdn.jsdelivr.net/gh/s9e/hljs-loader@1.0.34/loader.min.js|https://{domain}/hljs/loader.min.js|g" \
  repository.xml

# 移除 integrity 属性，本地文件哈希不一样
sed -i "s|<xsl:attribute name=\"integrity\">sha384-.\{64\}</xsl:attribute>||" repository.xml
```

改完之后需要重建 forum.js 把 XSL 模板重新编译进去。这次学乖了，`theme_secondary_color` 不敢碰（上次改坏过），改用 `theme_dark_mode` toggle：

```sql
-- 0→1 触发重建，访问一次站点后再 1→0 改回来
UPDATE settings SET value = '1' WHERE `key` = 'theme_dark_mode';
-- 访问站点触发 forum.js 重建 →
UPDATE settings SET value = '0' WHERE `key` = 'theme_dark_mode';
```

重建后 forum.js 里 0 个 jsdelivr，所有 hljs 引用指向本地域名。

### 又又又炸了：storage/formatter 缓存

以为这下终于完事了，结果改了 6 个文件、清了缓存、重建了 forum.js，刷新还是从 jsdelivr 加载。

我停用了浏览器缓存，重新看了下页面源码——服务器返回的 HTML 里 hljs-loader 的 `src` 还是 `cdn.jsdelivr.net`。说明问题**不在前端**，在服务器端渲染。

搜 `storage/` 目录：

```bash
grep -r "jsdelivr" storage/
```

```
storage/cache/77/e1/...
storage/formatter/Renderer_e77ed28....php
```

s9e 的 XSL 模板不是每次请求都解析源文件的——它编译完会缓存到 `storage/formatter/` 里。源文件改了 6 个，但 **s9e 运行时读的是这个编译缓存**，里面还是旧的 jsdelivr。

这就是为什么改了源文件、重建了 forum.js，还是从 jsdelivr 加载——forum.js 是前端，而 hljs-loader 的 `<script src>` 是 s9e **服务器端**在渲染帖子内容时写进 HTML 的。源文件 → 编译缓存 → 服务器端输出，我修了第一步，忘了第二步。

```bash
# 清掉 s9e 的编译缓存和 Laravel 缓存
sudo rm -rf storage/formatter/* storage/cache/* storage/views/*

# 重新 toggle 设置触发编译
# 访问站点 → s9e 从修复后的源文件重新编译 XSL 模板
```

清完缓存后再跑 `grep -r`：

```bash
grep -o "cdn.jsdelivr.net[^\"]*" storage/formatter/*.php
# 空——重新编译后没有 jsdelivr 了
grep -o "metazone.cc/hljs[^\"]*" storage/formatter/*.php
# metazone.cc/hljs/loader.min.js ✓
```

所以回头一看，这帖子里说的"改了 5 个文件"是骗人的，其实是 **6 个文件 + 1 层缓存**。发帖的时候信誓旦旦说全搞定了，结果马上就被打脸，紧接着又被缓存背刺。

改完之后老实跑三遍：

```bash
# 1. 源文件
grep -r "cdn.jsdelivr.net" vendor/ | grep -v "\.map\|locale/\|cdn.js"

# 2. 编译缓存
grep -r "cdn.jsdelivr.net" storage/

# 3. 前端产物
grep -r "cdn.jsdelivr.net" public/assets/
```

三层都过了才叫真改完了。别跟我一样以为自己改完了，过两天又炸。
