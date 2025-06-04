# GCP三种SSH登录方式详解及最佳实践

在Google Cloud Platform（GCP）中，SSH登录VM实例有三种主流方式，每种方式适用于不同的运维和安全场景。本文将详细介绍它们的原理、优缺点和适用场景，帮助你选择最合适的方案。



## 1. OS Login

### 工作原理

**OS Login**是GCP官方推荐的现代化SSH用户管理方式。启用后，GCP会将Linux系统用户和SSH密钥与Google账号绑定，实现“云上统一用户管理”。其核心原理如下：

* 用户的公钥绑定在Google账号（Profile）上，不再存放在VM或项目的metadata里；
* 通过IAM权限控制谁有权登录哪些实例；
* 登录时会自动为用户创建/同步对应的Linux账户。

### 优势

* **自动化同步**：用户入职、离职或权限变更时，无需逐台服务器操作，只要改动IAM权限即可。
* **统一审计**：结合Google Cloud Audit Log，可精确追踪谁登录过哪些机器。
* **支持多因子认证（MFA）**：提升安全性。
* **细粒度权限控制**：通过角色可以限制用户是否具备sudo权限等。

### 配置方式

1. **启用OS Login**

   * 针对VM或整个项目设置metadata：`enable-oslogin=TRUE`
2. **分配IAM角色**

   * 例如分配`roles/compute.osLogin`或`roles/compute.osAdminLogin`
3. **用户登录**

   * 用户使用`gcloud compute ssh`自动完成密钥推送和账号同步。

### 注意事项

* 首次登录会自动创建对应的Linux用户（如[user@google.com](mailto:user@google.com)→user\_google\_com）。
* 启用OS Login后，metadata的ssh-keys将不再生效（OS Login优先生效）。
* 需分配合适的IAM角色，普通登录和管理员（sudo）权限需不同角色。


**如果是临时登录，使用gcloud compute ssh命令后，会自动创建一个叫 【user】的用户名，而不是【user_google_com】,这样如果你的两个account的前缀一样，你的两个account会登录进相同的一个Linux账户，例如你的两个GCP acount分别是abcr@google.com和abc@gmail.com，那么你这两个account登录到vm后使用的Linux账号都是abc**



---

## 2. 管理Metadata SSH密钥

### 工作原理

这是传统方式，将SSH公钥直接写入到VM实例或项目级别的metadata（`ssh-keys`字段）。实例启动时，会自动将这些密钥同步到Linux用户的`~/.ssh/authorized_keys`中。

### 两种层级

* **实例级（Instance-level）**：只对特定VM生效；
* **项目级（Project-level）**：对项目下所有VM生效，优先级低于实例级。

### 优势

* **配置简单**，无需依赖IAM或Google账号体系；
* **兼容自定义工具和脚本**。

### 缺点

* **难以集中管理**，用户离职需手动逐台移除密钥；
* **安全隐患**，密钥失效和轮换需要额外人工操作；
* **不支持自动审计和权限细分**。

### 配置方法

1. 在实例或项目的metadata中添加ssh-keys，每行格式为：
   `username:ssh-rsa AAAAB3... user@example.com`
2. 用户使用对应私钥即可SSH登录。

### 实践建议

* 适合小型环境、临时测试，生产建议优先考虑OS Login。
* 密钥用完请及时删除。
* 可以通过`block-project-ssh-keys=TRUE`阻止项目级密钥。

---

## 3. 临时授权用户访问实例

### 工作原理

这种方式主要通过GCP控制台实现，适合临时运维、协助或紧急场景。
流程如下：

* 在GCP控制台VM详情页，通过“SSH”按钮或“Troubleshoot”面板选择“Add access”；
* 指定Google账号和访问时长（如1小时），系统自动生成密钥并推送到VM；
* 到期后密钥自动失效，无需人工回收。

### 优势

* **无需手动管理密钥**，临时授权结束后自动收回权限；
* **非常适合临时协助和运维支持场景**。

### 局限

* 只能通过GCP控制台进行，不支持批量自动化；
* 权限粒度较粗，只能指定账号和有效期。

### 使用方法

1. 进入GCP控制台的VM详情页，点击“SSH”→“Troubleshoot”→“Add access”；
2. 填写需要授权的Google账号和时长，提交后对方可在有效期内登录。

---

## 三种方式对比一览表

| 方式             | 管理维度         | 推荐场景      | 自动同步 | 支持审计 | 密钥时效 | 支持自动化 | 依赖IAM | 备注      |
| -------------- | ------------ | --------- | ---- | ---- | ---- | ----- | ----- | ------- |
| OS Login       | Google账号/IAM | 企业、多人、合规  | ✔️   | ✔️   | 支持   | ✔️    | ✔️    | 推荐生产使用  |
| Metadata SSH密钥 | VM/项目        | 小型、临时     | ❌    | ❌    | 长期   | ✔️    | ❌     | 需手动管理密钥 |
| 临时授权（Console）  | 控制台          | 临时协助/紧急场景 | ✔️   | 部分   | 短期   | ❌     | ✔️    | 自动失效    |

---

## 常用操作示例

### 启用OS Login

```bash
# 启用项目级OS Login
gcloud compute project-info add-metadata --metadata enable-oslogin=TRUE

# 分配IAM角色
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
  --member="user:YOUR_USER_EMAIL" \
  --role="roles/compute.osLogin"

# 用户登录
gcloud compute ssh YOUR_INSTANCE_NAME
```

### 添加Metadata SSH密钥

```bash
gcloud compute instances add-metadata INSTANCE_NAME \
  --metadata ssh-keys="user:ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQD... user@example.com"
```

### 控制台临时授权

* 按页面指示操作即可。

**这里需要注意，通过UI界面登录，其实相当于在GCP后台临时启动了一个proxy，然后临时生成了一对公私密钥，把公钥上传到目标VM的[username]/.ssh/authorized_keys，而且这个public key会被设定一个很短的过期时间，它只用在当前登录的时候给客户做认证。下次登录的时候这个key已经被删掉，会有新的public key被加入到authorized_key文件中。**


**临时登录也分两种情况，如果是用UI，那么就像上面说的那样，公钥和私钥都是临时生成的，但是如果是在cloud shell上的话，公私密钥都是会永久保存在cloud shell。**


---


### 在VM上获取metadata上面的ssh key信息

不知道为什么，无法在GCP的VM上更新metaserver上的hostkey，导致新的cloud shell在尝试登录的时候提示hostkey验证失败。

```
curl -H "Metadata-Flavor: Google"      "http://metadata.google.internal/computeMetadata/v1/instance/guest-attributes/hostkeys/ssh-ed25519?recursive"

```

手动更新hostkey

https://cloud.google.com/compute/docs/metadata/manage-guest-attributes#set_guest_attributes

```
curl -X PUT --data "AAAAC3NzaC1lZDI1NTE5AAAAIOXi38cYbYDsgl1vKRPMMGwtVDjIZ1lQgJLJPwtiTo+/" http://metadata.google.internal/computeMetadata/v1/instance/guest-attributes/hostkeys/ssh-ed25519 -H "Metadata-Flavor: Google"
```

更新完metadata的公钥成功登陆VM




为公钥生成指纹

```
ssh-keygen -lf <(echo "粘贴公钥内容")
```

VM上的hostkey在`/etc/ssh`目录下。


```
gcloud compute ssh --zone "us-central1-a" "curl-vm" --tunnel-through-iap --project "haoming-demo" --strict-host-key-checking=no
```

## 总结

GCP支持多种SSH登录机制，以满足不同规模和安全需求。
**企业和生产环境建议优先使用OS Login**，它结合了集中管理、自动化、审计和安全等优势。传统的Metadata SSH密钥方式适合临时、非生产环境。
临时授权方式则非常适合应急和协助场景，用完即失效，无需担心安全隐患。

通过合理选择和配置，可以大幅提升云上服务器的安全性与运维效率。

---

如需更详细的操作脚本或自动化方案，欢迎评论交流！
