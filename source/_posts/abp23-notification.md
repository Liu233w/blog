---
categories:
  - blog
date: 2018-04-02 10:52:38
tags:
- Web 开发
- AspNetBoilerplate
- Asp.Net

title: ABP 通知系统数据表结构及源码分析
---

# 前言
这篇文章接着 [在 ABP 2.3 和 Vue 单页应用中的踩坑总结](/2018/03/27/abp11/) 来写。

我当时在魔改 ABP 通知系统的时候读了一下它的源码，现在把总结写在这里。

注： 本文涉及的abp版本为 2.3.0，在更新的版本中可能会有出入。

# 类与表名的对应

ABP的通知Entity有4种类型（有趣的是这个不以Entity结尾），4个类分别对应数据库中的4张表：

|类(Entity)|表|说明|
|------------|
|NotificationInfo|AbpNotifications|发给单个用户的通知，里面包含了接收方用户的id|
|TenantNotificationInfo|AbpTenantNotifications|发送的某一类型的通知，只保存通知的内容，不保存用户的ID。注意，这个名字起得不大好，其实这个很租户关系不大|
|UserNotificationInfo|AbpUserNotifications|User和TenantNotification的映射，将这两个表关联起来，同时记录用户对订阅消息的状态（已读/未读）|
|NotificationSubscriptionInfo|AbpNotificationSubscriptions|记录了某某用户订阅了某某`NotificationName`或者`EntityTypeName`|

通知本身分为两种：
1. 向某用户单独发送的通知：这种通知只使用 `NotificationInfo` 来保存。在直接向某一个用户发送通知时，使用这张表来存储。
2. 采用“订阅、发布”模式发送的通知：用户订阅某种类型（或者某个实体）的通知时，首先把订阅信息存储在 `NotificationSubscriptionInfo` 中。当发布某个类型的通知时，此通知先被写入
   `TenantNotificationInfo`，然后ABP会根据`NotificationSubscriptionInfo`自动为每个订阅的用户在`UserNotificationInfo`中添加记录。通过`_userNotificationManager.GetUserNotificationsAsync`会根据`UserNotificationInfo`连接`TenantNotificationInfo`来获取到数据。

# 数据表说明
这里就光给我用到的字段添加注释了

## AbpNotifications
|列名|类型|注释|
|-------|
|Id|int||
|CreationTime|datetime2(7)||
|CreatorUserId|bigint||
|Data|nvarchar(MAX)||
|DataTypeName|nvarchar(512)||
|EntityId|nvarchar(96)||
|EntityTypeAssemblyQualifiedName|nvarchar(512)||
|EntityTypeName|nvarchar(250)||
|ExcludedUserIds|nvarchar(MAX)||
|NotificationName|nvarchar(96)||
|Severity|tinyint||
|TenantIds|nvarchar(MAX)||
|UserIds|nvarchar(MAX)||

## AbpNotificationSubscriptions
|列名|类型|注释|
|------|
|Id|uniqueidentifier||
|CreationTime|datetime2(7)||
|CreatorUserId|bigint|Creator保存的是当前 Session中的用户ID，UserId是在NotificationSubscriptionManager中指定的用户ID。 如果是管理员让某个用户订阅的话，这两个ID就不一样了。|
|EntityId|nvarchar(96)||
|EntityTypeAssemblyQualifiedName|nvarchar(512)||
|EntityTypeName|nvarchar(250)||
|NotificationName|nvarchar(96)|消息名，用这个来标识消息的分类|
|TenantId|int||
|UserId|bigint|订阅者的UserId|

## AbpTenantNotifications
|列名|类型|注释|
|------|
|Id|bigint||
|CreationTime|datetime2(7)||
|CreatorUserId|bigint||
|Data|nvarchar(MAX)|根据DataTypeName动态改变|
|DataTypeName|nvarchar(512)||
|EntityId|nvarchar(96)||
|EntityTypeAssemblyQualifiedName|nvarchar(512)||
|EntityTypeName|nvarchar(250)||
|NotificationName|nvarchar(96)||
|Severity|tinyint||
|TenantId|int||

## AbpUserNotifications
|列名|类型|注释|
|------|
|Id|uniqueidentifier||
|CreationTime|datetime2(7)||
|State|int|状态|
|TenantId|int||
|TenantNotificationId|uniqueidentifier|订阅消息ID|
|UserId|bigint|用户ID|

# 接口说明
- 通过 UserNotificationManager 获得的是 UserNotification，访问其 Notification 属性才能得到 TenantNotification
- 通过 UserNotificationManager 输入 id 来获取notification时，需要输入 UserNotification 的id
- 租户：UserNotification需要输入 UserIdentifier，这个里面的租户id只用于 UnitOfWork 的 SetTenantId，用来筛选上述的三种通知

# 相关源码
- https://github.com/aspnetboilerplate/aspnetboilerplate/blob/f10fa5205c780bcc27adfe38aaae631f412eb7df/src/Abp/Notifications/UserNotificationManager.cs
- https://github.com/aspnetboilerplate/aspnetboilerplate/blob/bc266e3503fcb2825684ec829cc356815f94f397/src/Abp.Zero.Common/Notifications/NotificationStore.cs
- https://github.com/aspnetboilerplate/aspnetboilerplate/blob/f10fa5205c780bcc27adfe38aaae631f412eb7df/src/Abp/Notifications/UserNotificationInfoExtensions.cs
- https://github.com/aspnetboilerplate/aspnetboilerplate/blob/f10fa5205c780bcc27adfe38aaae631f412eb7df/src/Abp/Notifications/UserNotificationInfoWithNotificationInfoExtensions.cs
- https://github.com/aspnetboilerplate/aspnetboilerplate/blob/f10fa5205c780bcc27adfe38aaae631f412eb7df/src/Abp/Notifications/UserNotificationInfoWithNotificationInfo.cs
