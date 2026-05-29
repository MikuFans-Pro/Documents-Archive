# Flarum 之九 - 之八补遗：第21个坑，integrity hash 死灰复燃

> 导读：本文是「Flarum部署连环坑」系列的一部分。完整的问题梳理与解决方案总结，请参阅：[Flarum之八--20坑复盘与国内部署最小踩坑路径（最终章？）](https://lab.metazone.cc/d/13)

## 现象

两个站点的 console 同时亮了红——`loader.min.js` 被浏览器直接拦截。lab 和 acg 一起炸，acg 还多了一个 emoji TypeError。

lab：

```
Failed to find a valid digest in the 'integrity' attribute for resource
'https://lab.metazone.cc/hljs/loader.min.js' with computed SHA-384 integrity
'rEQib25mSC4Y0Ln3fJPWlc0qK5/+ENy5lytQlo0x4Vrdt41iCxaGUoRMPJmWbaVJ'.
The resource has been blocked.
```

acg 除了同样的 integrity 拦截，还有：

```
cdn.ts:3 Uncaught TypeError: Cannot read properties of null (reading '1')
    at renderEmoji.ts:48:5
```

## 问题一：integrity hash 拦截

浏览器加载 `loader.min.js` 时，下载完算了真实 hash，跟 HTML 里 `integrity` 属性声明的不一样，就直接拦截。

这个坑[之三](https://lab.metazone.cc/d/7)修过：

```bash
# 之三当时写的（Flarum 1.x）
sed -i "s|<xsl:attribute name=\"integrity\">sha384-.\{64\}</xsl:attribute>||" repository.xml
```

但之六升级到 2.0 后，s9e 换了新版 hljs-loader，`repository.xml` 也被覆盖了。之六的 URL 替换做了——`src` 指向了本地 `hljs/loader.min.js`——但 integrity 属性没再删一次。还是 jsdelivr 原始文件的 sha384。

而本地的 `loader.min.js` 是打过补丁的（fallback URL 自动推导 + 竞态修复），文件内容变了，hash 也不一样。浏览器一对比，对不上，拦截。

四篇文章的时间线一目了然：

```
之三（1.x）: 改 URL + 删 integrity → 有效 ✅
之六（2.0）: 升级后 repository.xml 被覆盖
            URL → 本地 ✅
            integrity → 忘了删 ✖️
```

之六改完的 repository.xml 长这样：

```xml
<xsl:if test="...">
  <xsl:attribute name="integrity">sha384-rEQib25mSC4Y0Ln3fJPWlc0qK5/+ENy5lytQlo0x4Vrdt41iCxaGUoRMPJmWbaVJ</xsl:attribute>
  <xsl:attribute name="src">https://acg.metazone.cc/hljs/loader.min.js</xsl:attribute>
</xsl:if>
```

src 对了，integrity 没变。之八列了 20 个坑、画了四层模型、给了最小化踩坑路线，但那条路线里没说"升级后如果 repository.xml 更新了，integrity 要重新移除"。漏一步，白干。

## 问题二：acg 的 emoji TypeError

这也是一个回来了的坑。之六 C5 写过——`@twemoji/api` 包的 `w.base` 被 sed 全局替换误伤了。

编译后的 `forum.js` 里有：

```javascript
const D = /([0-9]+)\.[0-9]+\.[0-9]+/g.exec(w.base)[1];
```

`w.base` 是从 twemoji CDN URL（`jdecked/twemoji@17.0.2/assets/`）里提取版本号的。sed 做 CDN 本地化替换时，把这个 URL 换成了不带版本号的本地路径，正则匹配不到数字，`exec` 返回 null，`[1]` 直接炸。

之六的修复是 hardcode 版本号——把那一行替换为 `"17"`。

但 forum.js 是编译产物。修 integrity 的时候清了 formatter 缓存、触发了热编译，forum.js 被重新从源码生成了，hardcode 被冲掉，错误又回来了。

## 修复

两个事一起办了。

```bash
# 1. 删 integrity（两个站点）
cd /var/www/lab_metazone_cc
sed -i "s|<xsl:attribute name=\"integrity\">sha384-.\{64\}</xsl:attribute>||" \
  vendor/s9e/text-formatter/src/Plugins/BBCodes/Configurator/repository.xml

cd /var/www/acg_metazone_cc
sed -i "s|<xsl:attribute name=\"integrity\">sha384-.\{64\}</xsl:attribute>||" \
  vendor/s9e/text-formatter/src/Plugins/BBCodes/Configurator/repository.xml

# 2. 清缓存 + 触发 forum.js 重建
rm -rf storage/formatter/*
php flarum cache:clear
curl -s https://lab.metazone.cc/ > /dev/null
curl -s https://acg.metazone.cc/ > /dev/null

# 3. acg 的 forum.js hardcode 版本号
python3 -c "
path = '/var/www/acg_metazone_cc/public/assets/forum.js'
with open(path) as f: content = f.read()
content = content.replace(
    '/([0-9]+)\\\\.[0-9]+\\\\.[0-9]+/g.exec(w.base)[1]',
    '\"17\"'
)
with open(path, 'w') as f: f.write(content)
"

sudo systemctl restart php-fpm
```

验证：

```bash
# 两个站 repository.xml 都不该有 integrity
grep -c "integrity" vendor/s9e/text-formatter/src/Plugins/BBCodes/Configurator/repository.xml
# 期望：0

# acg forum.js 里版本号写死为 "17"
grep -c '"17"' public/assets/forum.js
# 期望：1
```

## 新增第21坑

**C10 —— integrity hash 死灰复燃**

```
严重度  : 🔴 loader.min.js 被浏览器拦截，代码高亮完全挂掉
原因    : 大版本升级后 repository.xml 被覆盖，URL 替换做了，integrity 没重新移除。
          本地 loader.min.js 打过补丁，文件内容变了，hash 跟原始不匹配，浏览器 SRI 拦截。
知道就行 : 升级后 CDN 本地化要改 URL + 删 integrity，少一步白干
```

## 教训

SRI 在校验 CDN 文件时提供安全，但在 CDN 本地化场景下完全是反作用——你改了文件内容，它就不让你加载。本地化的文件，integrity 必须移除。

hardcode 编译产物 + 热编译触发是个不稳的组合。forum.js 的 `w.base` 修复直接写在产物里，一重建就丢。更稳的做法是从 s9e 源码层面把版本号提取干掉，但改动更大，暂时不碰。

另外，坑是真的会回来的。写完 20 坑以为自己已经列清楚了，结果部署流程里少删一个 integrity 属性、触发了重编译冲掉 hardcode，之前修过的东西又回来了。

>! 累了，毁灭吧，不知道还有多少坑，严重怀疑这个系列可以写一本书... !<