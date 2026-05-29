# Flarum 之七 - disable-email-notifications 扩展适配 Flarum 2.0

> 导读：本文是「Flarum部署连环坑」系列的一部分。完整的问题梳理与解决方案总结，请参阅：[Flarum之八--20坑复盘与国内部署最小踩坑路径（最终章？）](https://lab.metazone.cc/d/13)

ACG 站升级到 2.0 RC1 之后，准备把全部踩过的坑捋一遍然后推 lab。其中一个必须解决的问题是 `metazone/disable-email-notifications`——这个扩展在 lab 上是启用的，升级不能直接移除，得适配一下。

## 扩展做了什么

逻辑很简单，两件事：

1. **新用户注册时**：监听 `Registered` 事件，把该用户所有通知类型的邮件偏好关掉
2. **全局邮件驱动层**：从容器里直接移除 `flarum.notification.drivers` 里的 `email` 键，确保即使有新的通知类型也不会有邮件通知

不依赖后台设置、不依赖用户操作，从驱动层一锤定音。

## 2.0 API 变化分析

```
ExtenderInterface  1.x: extend(Container $c, Extension $ext = null)
                   2.0: extend(Container $c, ?Extension $ext = null): void
                       ↑ 加了 nullable 类型提示 + void 返回类型

Extend\Event       1.x: listen($event, $listener)
                   2.0: listen(string $event, callable|string $listener): self
                       ↑ 签名更严格，但调用方式完全相同 ✅

Registered         1.x: __construct(User $user)
                   2.0: __construct(User $user, ?User $actor = null)
                       ↑ 多了 $actor 参数，$event->user 照用 ✅

flarum.notification.drivers    不变 ✅
flarum.notification.blueprints 不变 ✅
User::getNotificationPreferenceKey 不变 ✅
User::setPreference            不变 ✅
```

结论：**这个扩展的 Flarum API 依赖纹丝不动**，唯一需要改的是 `ExtenderInterface` 的签名细节。

## 改动

**只改了两行：**

```diff
# composer.json
- "flarum/core": "^1.0"
+ "flarum/core": "^1.0|^2.0"

# src/RemoveEmailDriver.php
- public function extend(Container $container, Extension $extension = null): void
+ public function extend(Container $container, ?Extension $extension = null): void
```

`DisableEmailNotificationsListener` 和 `extend.php` 完全不用动——它们用的 `Event`、`Registered`、`User` 在 2.0 里接口没变。

发布版本：**v1.1.0**（MINOR 版本号升级，向后兼容，无破坏性变更）。

## 兼容性

```
Flarum 1.x  ✅  ^1.0 约束覆盖，原有逻辑不变
Flarum 2.0  ✅  已在 RC1 环境验证通过
```

两处改动都是**向后兼容**的——`^1.0|^2.0` 约束在 1.x 和 2.0 下都能解析，`?Extension` 语法 PHP 7.1+ 就支持了，1.x 和 2.0 都跑在 PHP 8+。

## 验证

在 ACG 站（2.0 RC1）上完整流程：

1. 更新 `composer.json` 约束 + `RemoveEmailDriver` 签名
2. 通过 Composer path repo 安装
3. `php flarum extension:enable` 启用
4. `cache:clear` + 重启 PHP-FPM
5. 站点 HTTP 200，无 PHP 报错，无日志异常

lab 升级时只需在升级前更新 git 仓库引用，升级后 `composer update` 自然拉取新版即可。

## 教训

适配 2.0 不一定需要大改。这个扩展碰巧只依赖了 Flarum 中最稳定的几个接口（Event 系统、容器绑定、User 偏好），这些在 2.0 里都没变。真正容易出问题的是语言包、前端 JS 编译、权限数据迁移——那些才是要重点关注的。
