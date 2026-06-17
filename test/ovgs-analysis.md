# OVGS 深入分析报告

> 基于 `ovgs.proto` 与 `docs/service-overview.md` 综合分析

---

## 一、核心定位

OVGS（Ownership Voucher gRPC Service）是一个基于 gRPC 的服务，允许组织管理员管理用户和设备，并使用户能够为被分配的设备请求 **RFC8366 所有权凭证（Ownership Voucher）**。其核心目的是：让用户为所属组织拥有的网络设备组件（通过序列号 + IEN 标识）请求所有权凭证。

- 协议版本：`proto3`
- 包名：`ovgs.v1`
- Go 包路径：`github.com/aristanetworks/ownership-voucher-grpc/ovgs`
- 许可证：Apache 2.0（Arista Networks, Inc.）

---

## 二、认证与授权体系

### 2.1 认证

- 假定由供应商提供，示例基于 OIDC 认证
- 用户/服务账号的创建均在 OVGS 外部完成，OVGS 仅处理角色分配

### 2.2 授权模型

- 基于组织树（Organization Tree）的层级授权
- **权限继承**：用户可访问其所属组及所有子组（子树），子组无法访问父组信息
- **权限叠加**：当用户属于多个子树且存在重叠时，取最高权限
- 用户唯一标识：`(username, user_type, org_id)` 三元组

---

## 三、四级角色体系

### 3.1 角色定义（Proto Enum）

```protobuf
enum UserRole {
  USER_ROLE_UNSPECIFIED = 0;
  USER_ROLE_SUPPORT     = 1;  // 供应商内部运维
  USER_ROLE_ADMIN       = 2;  // 组织管理员
  USER_ROLE_ASSIGNER    = 3;  // 资源分配者
  USER_ROLE_REQUESTOR   = 4;  // 只读请求者
}
```

### 3.2 角色权限矩阵

| 角色 | 权限范围 | 可执行 RPC | 可被谁分配/移除 |
|------|---------|-----------|----------------|
| **SUPPORT** | 全部 RPC；委派管理仅限根组 owner org ADMIN | 全部 | 同组/父组的 SUPPORT |
| **ADMIN** | 同 SUPPORT；委派管理仅限根组 owner org ADMIN | 全部 | 同组/父组的 SUPPORT/ADMIN |
| **ASSIGNER** | 除创建/删除组、用户角色管理、委派组织管理外的所有 RPC | CreateDomainCert, AddSerial, RemoveSerial, GetOwnershipVoucher 等 | 同组/父组的 SUPPORT/ADMIN |
| **REQUESTOR** | 只读 + 请求所有权凭证 | GetGroup, GetUserRole, GetDomainCert, GetSerial, GetOwnershipVoucher | 同组/父组的 SUPPORT/ADMIN/ASSIGNER |

### 3.3 角色分配约束（Proto 明确定义）

```
Caller Role    |   Assignable Roles
 ADMIN         |    ADMIN, ASSIGNER, REQUESTOR
 ASSIGNER      |    ASSIGNER, REQUESTOR
 REQUESTOR     |    NA (cannot assign roles)
```

> **文档仅说 ADMIN 可调用 AddUserRole，Proto 实际允许 ASSIGNER 分配 ASSIGNER/REQUESTOR 角色。**

---

## 四、核心数据模型

### 4.1 实体关系图

```
Organization (root group, group_id = org_id)
  ├── Group (child, inherits delegated_orgs from parent)
  │     ├── Component (ien + serial_number)
  │     │     └── TpmInfo { ek_data: endorsement_key | ek_certificate, platform_primary_key }
  │     ├── DomainCert (cert_id, certificate_der, revocation_checks, expiry_time)
  │     ├── User (username, user_type, org_id, user_role)
  │     ├── delegated_orgs: map<org_id, GroupList>
  │     └── Child Groups (recursive)
  └── All Serial Numbers (default allocation by vendor)
```

### 4.2 Component — 设备组件标识

```protobuf
message Component {
  string ien = 1;           // IANA Enterprise Number (如 30065 = Arista)
  string serial_number = 2; // 设备序列号
}
```

- IEN 是厂商标识，与序列号组合才能全球唯一
- OVGS 天然支持**多厂商设备**共存于同一服务

### 4.3 TpmInfo — TPM 多版本支持

```protobuf
message TpmInfo {
  oneof ek_data {
    bytes endorsement_key    = 1;  // TPM 1.2：仅背书密钥公钥
    bytes ek_certificate     = 2;  // TPM 2.0 + IDevID：完整 X.509 背书证书
  }
  bytes platform_primary_key = 3;  // TPM 2.0 无 IDevID：平台主密钥公钥
}
```

| 字段 | 适用场景 | TPM 版本 |
|------|---------|---------|
| `endorsement_key` | 仅背书密钥公钥 | TPM 1.2 |
| `ek_certificate` | 完整 X.509 背书证书 | TPM 2.0 + IDevID |
| `platform_primary_key` | 平台主密钥公钥 | TPM 2.0 无 IDevID |

> `public_key_der` 已标记 `deprecated = true`，统一迁移至 `TpmInfo` 结构。

### 4.4 AccountType — 账号类型

```protobuf
enum AccountType {
  ACCOUNT_TYPE_UNSPECIFIED    = 0;
  ACCOUNT_TYPE_USER           = 1;  // SSO 用户账号
  ACCOUNT_TYPE_SERVICE_ACCOUNT = 2; // 服务账号（程序化访问）
}
```

### 4.5 GetGroupResponse — 组完整视图

```protobuf
message GetGroupResponse {
  string group_id              = 1;
  repeated string cert_ids     = 2;
  repeated Component components = 3;
  repeated User users          = 4;
  repeated string child_group_ids = 5;
  string description           = 6;
  map<string, GroupList> delegated_orgs = 7;  // 通配符 "*" 允许任意组织
}
```

---

## 五、跨组织委派机制

### 5.1 核心概念

- 一个组可以被委派给另一个组织，需先将该组织加入组的 **允许委派组织列表**
- 只有 **owner org 的根组 ADMIN** 可执行委派组织管理（AddGroupDelegatedOrg / RemoveGroupDelegatedOrg）
- **新建组自动继承父组的 delegated_orgs**
- 支持通配符 `"*"`，表示允许任意组织委派访问
- **禁止自委派**：`org_id` 不能等于组所属组织

### 5.2 委派安全边界

| 操作 | 本组织调用者 | 委派调用者 |
|------|------------|-----------|
| **AddUserRole** | 可添加任何用户（需委派组织在 delegated_orgs 中） | 只能添加**本组织**用户 |
| **RemoveUserRole** | 可移除任何用户 | 只能移除**本组织**用户 |
| **GetGroup** | 返回所有用户 + 所有委派组织 | 仅返回本组织用户 + 仅显示本组织委派信息 |
| **GetUserRole** | 可查看同组用户角色 | **无法查看** owner org 用户的角色 |

### 5.3 删除委派组织的前置条件

所有来自该委派组织的用户必须已从所有相关组/子组中移除，且该组是唯一启用该组织委派的源组时，才允许删除。

---

## 六、gRPC API 完整一览（17 个 RPC）

### 6.1 组管理

| RPC | 方法类型 | 最低角色 | 功能 | 关键错误 |
|-----|---------|---------|------|---------|
| `CreateGroup` | Unary | ADMIN | 在现有组或组织下创建子组（继承委派、同父组禁止重名） | INVALID_ARGUMENT, NOT_FOUND, ALREADY_EXISTS |
| `GetGroup` | Unary | REQUESTOR | 查看组的子组、证书ID、序列号、用户映射、委派组织 | NOT_FOUND, PERMISSION_DENIED |
| `DeleteGroup` | Unary | ADMIN | 删除空组（无子组/证书/序列号/用户时才允许，根组不可删） | INVALID_ARGUMENT, NOT_FOUND, FAILED_PRECONDITION |

### 6.2 委派管理

| RPC | 方法类型 | 最低角色 | 功能 | 关键错误 |
|-----|---------|---------|------|---------|
| `AddGroupDelegatedOrg` | Unary | 根组 Owner Org ADMIN | 添加允许委派的组织（支持通配符 `"*"`） | INVALID_ARGUMENT, NOT_FOUND, ALREADY_EXISTS |
| `RemoveGroupDelegatedOrg` | Unary | 根组 Owner Org ADMIN | 移除委派组织（须先移除该组织所有委派用户） | INVALID_ARGUMENT, NOT_FOUND, FAILED_PRECONDITION |

### 6.3 用户角色管理

| RPC | 方法类型 | 最低角色 | 功能 | 关键错误 |
|-----|---------|---------|------|---------|
| `AddUserRole` | Unary | ADMIN（ASSIGNER 可分配 ASSIGNER/REQUESTOR） | 为组中的用户分配角色 | INVALID_ARGUMENT, NOT_FOUND, ALREADY_EXISTS |
| `RemoveUserRole` | Unary | ADMIN | 移除用户在组中的角色（即撤销访问） | INVALID_ARGUMENT, NOT_FOUND |
| `GetUserRole` | Unary | REQUESTOR | 查看用户在各组中的角色 | INVALID_ARGUMENT, NOT_FOUND |

### 6.4 域证书管理

| RPC | 方法类型 | 最低角色 | 功能 | 关键错误 |
|-----|---------|---------|------|---------|
| `CreateDomainCert` | Unary | ASSIGNER | 创建 PDC（证书 + 吊销检查 + 过期时间，三元组去重） | INVALID_ARGUMENT, NOT_FOUND, ALREADY_EXISTS |
| `GetDomainCert` | Unary | REQUESTOR | 查看 PDC 详情 | NOT_FOUND |
| `DeleteDomainCert` | Unary | ASSIGNER | 删除域证书 | NOT_FOUND |

### 6.5 序列号管理

| RPC | 方法类型 | 最低角色 | 功能 | 关键错误 |
|-----|---------|---------|------|---------|
| `AddSerial` | Unary | ASSIGNER | 将序列号分配给组（自动从原组移除，始终属于根组） | INVALID_ARGUMENT, NOT_FOUND, ALREADY_EXISTS |
| `RemoveSerial` | Unary | ASSIGNER | 从组中移除序列号 | INVALID_ARGUMENT, NOT_FOUND |
| `GetSerial` | Unary | REQUESTOR | 查询序列号详情（TPM信息、MAC地址、型号等） | INVALID_ARGUMENT, NOT_FOUND |
| `GetSerials` | Unary | — | 批量查询多个序列号（响应保持请求顺序） | 同 GetSerial |
| `GetComponents` | **Server Streaming** | REQUESTOR | 流式返回指定组下所有组件及序列详情 | NOT_FOUND |

### 6.6 所有权凭证签发

| RPC | 方法类型 | 最低角色 | 功能 | 关键错误 |
|-----|---------|---------|------|---------|
| `GetOwnershipVoucher` | Unary | REQUESTOR | 签发单个设备的 RFC8366 所有权凭证 | INVALID_ARGUMENT, FAILED_PRECONDITION |
| `GetOwnershipVouchers` | Unary | REQUESTOR | 批量签发多个设备的所有权凭证 | 同 GetOwnershipVoucher |

---

## 七、所有权凭证签发流程（核心功能）

### 7.1 `GetOwnershipVoucher` 执行逻辑

```
请求 → 验证权限 → 验证时效 → 验证组织一致性 → 签发凭证 → 返回 TPM 信息
```

1. **验证请求者权限**：确认请求者有权访问该序列号对应的设备
2. **验证时效**：请求的生命期不能在过去，且须在证书过期时间之内
3. **验证组织一致性**：证书和序列号必须属于同一组织（`org of cert == org of serial`）
4. **签发凭证**：使用指定参数生成 RFC8366 格式的 CMS 签名凭证
5. **返回 TPM 信息**：如可用，附带设备 TPM 背书密钥/证书，供客户做额外验证

### 7.2 请求参数

```protobuf
message GetOwnershipVoucherRequest {
  Component component       = 1;  // ien + serial_number
  string cert_id            = 2;  // 域证书 ID
  google.protobuf.Timestamp lifetime = 3;  // 凭证有效期限
}
```

### 7.3 响应内容

```protobuf
message GetOwnershipVoucherResponse {
  bytes voucher_cms    = 1;  // CMS 二进制格式凭证 (RFC5652)
  bytes public_key_der = 2;  // [deprecated] TPM 公钥
  TpmInfo tpm_info     = 3;  // TPM 详细信息
}
```

### 7.4 凭证格式（RFC8366 §5.3）

```json
{
  "ietf-voucher:voucher": {
    "created-on": "2023-02-11T13:45:31.69401473+05:30",
    "expires-on": "2023-08-11T13:45:31+05:30",
    "serial-number": "JPEXXXX27",
    "assertion": "verified",
    "pinned-domain-cert": "MIIFjjC...HBUoCj0M6oIjhTcvHQ==",
    "domain-cert-revocation-checks": true
  }
}
```

- `created-on` / `expires-on`：RFC3339 格式
- `pinned-domain-cert`：ASN.1 DER 编码的 X.509 证书，Base64 字符串
- 凭证以 CMS 二进制格式签名（不加密），仅由厂商签名
- **相同参数每次签发生成不同凭证，均为有效凭证**

---

## 八、AddSerial 双重权限校验

Proto 揭示了一个关键安全逻辑：

```
移动序列号时，调用者必须同时拥有源组和目标组的权限。
如果序列号仅在根组（未被分配到任何子组），则还需根组权限。
```

序列号归属规则：
- 始终属于根组
- 最多额外分配给一个子组
- 重新分配时自动从原子组移除

---

## 九、错误体系完整定义

| 错误码 | 触发场景 | 涉及 RPC |
|--------|---------|---------|
| `INVALID_ARGUMENT` | 空字段、过期时间、无效证书、IEN 不适用、重名组、禁止自委派 | 几乎所有 RPC |
| `NOT_FOUND` | 组/组件/证书/用户不存在 | 几乎所有 RPC |
| `ALREADY_EXISTS` | 组重名、用户已存在、证书三元组重复、委派组织已存在 | CreateGroup, AddUserRole, CreateDomainCert, AddGroupDelegatedOrg |
| `FAILED_PRECONDITION` | 删除非空组、组件/证书不存在时请求 OV、委派组织有残留用户 | DeleteGroup, GetOwnershipVoucher, RemoveGroupDelegatedOrg |
| `PERMISSION_DENIED` | 无权限访问 | 所有 RPC |

---

## 十、文档与 Proto 差异/矛盾点

| # | 差异 | 文档 | Proto |
|---|------|------|-------|
| 1 | RPC 总数 | 14 个 | **17 个**（多 GetComponents/GetSerials/GetOwnershipVouchers） |
| 2 | ASSIGNER 可分配角色 | 仅 ADMIN 可 AddUserRole | **ASSIGNER 也可分配 ASSIGNER/REQUESTOR** |
| 3 | 委派通配符 `"*"` | 未提及 | **支持 `"*"` 允许任意组织委派** |
| 4 | 新组继承委派 | 未提及 | **自动继承父组 delegated_orgs** |
| 5 | 组名唯一性 | 未提及 | **同父组下 description 唯一** |
| 6 | 禁止自委派 | 未提及 | **org_id 不能等于组所属组织** |
| 7 | SUPPORT 角色 | 表格中提到但未详细定义 | Proto 中 `USER_ROLE_SUPPORT = 1`，供应商内部运维 |
| 8 | CreateDomainCert 去重 | 未提及 | **相同 (certificate_der, revocation_checks, expiry_time) 三元组不可重复创建** |
| 9 | GetComponents 流式 API | 未提及 | **服务端流式 RPC，按组返回所有组件** |
| 10 | DeleteGroup 禁删根组 | 未提及 | **INVALID_ARGUMENT if group_id = root group** |

---

## 十一、架构设计要点总结

1. **树状多租户模型**：组织 → 根组 → 子组（单父多子），天然映射企业组织架构
2. **最小权限原则**：四级角色 + 子树隔离 + 委派边界，确保用户只能操作其可见范围内的资源
3. **序列号双重归属**：始终属于根组 + 最多一个子组，便于全局盘点又支持局部分配
4. **PDC 不继承**：域证书绑定到组而不下传子组，同一证书可手动关联多个组
5. **凭证签发幂等性**：相同参数每次签发生成不同凭证，均为有效凭证（CMS 签名含时间戳等因素）
6. **批量/流式支持**：`GetComponents`（流式）、`GetSerials`/`GetOwnershipVouchers`（批量），适配大规模设备管理场景
7. **TPM 渐进演进**：`public_key_der` → `TpmInfo`，支持 TPM 1.2/2.0 + IDevID 多种硬件信任锚
8. **委派继承 + 通配符**：新建组自动继承委派策略，`"*"` 通配符允许开放委派，兼顾安全与灵活性
9. **用户创建外部化**：OVGS 不负责用户创建，仅处理角色分配，与供应商身份管理系统解耦
10. **序列号生命周期由供应商管理**：采购、RMA、EFT、资产转移等均由供应商在根组操作
