# Kingbase 表结构对齐与 UUIDv7 实施计划

## 目标

以当前用户编辑的 `src/main/resources/db/migration/kingbase/V1__init_schema.sql` 中定义的表和列为唯一 schema 来源，仅保留有表支撑的功能，将所有关系记录/关系 ID 对齐为 UUIDv7，并最后追加 `sys_action` 初始化数据迁移。

## 当前阶段

阶段 3：实现

## 各阶段

### 阶段 1：现状与迁移盘点
- [x] 核验工作区改动，确认用户对 V1 的修正及新增 `sys_dict_type` / `sys_dict_datum` 表
- [x] 盘点仍映射缺失表的实体、仓库、服务和控制器
- [x] 盘点当前 schema 与剩余实体/API/查询差异
- [x] 确认 UUIDv7 采用 MyBatis-Plus `IdentifierGenerator` 自定义实现
- **状态：** complete

### 阶段 2：方案确认与工作计划
- [x] 确认只保留迁移中有对应表的模块
- [x] 确认根节点哨兵使用固定 UUIDv7 `00000000-0000-7000-8000-000000000000`
- [x] 确认动作权限种子迁移最后创建
- [x] 将用户确认的方案细化为此实施计划
- **状态：** complete

### 阶段 3：实现
- [ ] 引入 uuid-creator 并接入 MyBatis-Plus 自定义 UUIDv7 `IdentifierGenerator`，让自动填充与显式创建都统一生成字符串 UUIDv7
- [ ] 为保留的 DAO、外键、Repository/Service/API IDs 改用 `String`，移除 `SnowflakeHolder`/`SnowflakeIdGenerator`
- [ ] 统一菜单/动作父级 UUIDv7 根哨兵；在新增 V2 迁移中改写菜单默认值且保持用户 V1 修改不受影响
- [ ] 对齐 sys 用户、角色、菜单、动作、字典、权限关系、清理配置、报表、媒体资产、集群、会话、审计映射及查询；移除未迁移表的机制/知识/资源模块和媒体任务表逻辑
- [ ] 移除不存在的设备/平台持久化维度及其授权绑定；保持外部协议设备标识原有类型和职责
- [ ] 最后依据保留代码中的实际权限码创建可重复执行的 sys_action UUIDv7 初始数据迁移
- **状态：** in_progress

### 阶段 4：测试与验证
- [ ] 添加/调整 UUIDv7 生成器、ID 填充、根哨兵和表映射相关测试
- [ ] 使用 Java 25 尝试 Maven 编译/测试；如构建工具或运行条件不足，说明实际完成的替代验证
- [ ] 静态扫描 `@TableName` 表及持久化列、删除的模块引用、Snowflake 持久 ID 残留、迁移动作码一致性
- [ ] 运行 `git diff --check`，确保用户对 V1 的修改保留
- **状态：** pending

### 阶段 5：交付
- [ ] 汇总下线模块、UUIDv7 接入方案、迁移脚本和验证限制
- [ ] 检查最终工作区状态，不覆盖用户修改
- **状态：** pending

## 关键决策

| 决策 | 理由 |
|------|------|
| 当前工作区 `V1__init_schema.sql` 是权威 schema；包含用户刚加入的字典表 | 不回退、不覆盖用户维护的数据库结构 |
| UUID 使用字符串表示以匹配 Kingbase `VARCHAR(64)`，MyBatis-Plus 通过 `IdentifierGenerator.nextId` 生成 UUIDv7 | 统一手动赋值、元对象处理器和普通 MP 插入的 ID 行为 |
| 保留迁移外部设备/协议业务 ID 原样；只将关系表实体 ID 和实际关联字段改为 UUID 字符串 | 非本 schema 生成的领域标识不属于关系记录 UUID 主键 |
| sys_action 初始化数据脚本在代码/schema 适配结束后最后新增 | 权限种子应由最终保留的 Controller/Service 权限要求推导 |
| `V1` 是 Flyway 初始脚本；修改其结构会改变尚未部署环境的初始化，新增根哨兵变更放 `V2` | 保留既有编辑，避免改变 V1 的校验校验和 |

## 关键发现

- V1 表集合包括 `sys_user`, `sys_role`, `sys_menu`, `sys_action`, 三类用户/角色授权关联、`sys_storage_cleaner_config`, `sys_dict_type`, `sys_dict_datum`, `report_execution_record`, `media_asset`, 三张集群表、刷新令牌、访问审计日志。
- V1 未定义设备/平台字段；菜单及动作根 PID 当前约定 0/-1；V1 菜单父 ID 默认仍为字符列默认值 `0`。
- 缺表实体：`sys_mechanism*`, `sys_knowledge`, `sys_resource_*`, `media_transcode_task`。
- MyBatis-Plus 3.5.17 的 `IdentifierGenerator` 提供 `Number nextId(Object)` 与默认 `String nextUUID(Object)`；`IdType.ASSIGN_UUID` 将调用 `nextUUID`。
- V1 字典两张表尚没有主键/唯一/外键约束，当前代码需保持与现有表定义一致，不把约束自行加回去。

## 遇到的错误

| 错误 | 尝试次数 | 解决方案 |
|------|---------|---------|
| 初次定位 MyBatis-Plus 类时选中 `-sources.jar` 而非普通 jar | 1 | 使用本地 `mybatis-plus-core-3.5.17.jar` 用 Java 25 `javap` 确认接口签名 |

## 备注

- 当前 Maven wrapper、系统 Maven 与 psql 未发现；Java 25 已安装。
- 每个阶段结束更新 `progress.md` 与本计划状态。
