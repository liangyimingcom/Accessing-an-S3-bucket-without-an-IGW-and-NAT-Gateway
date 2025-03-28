# 在没有IGW和NAT Gateway的情况下访问S3桶（公开桶设置白名单/或者私有桶需要配置IAM忽略）

## 1、搭建EC2的测试环境：在没有IGW和NAT Gateway的情况下使用Session Manager连接实例

是的，即使没有Internet Gateway (IGW)或NAT Gateway，您也可以成功地使用AWS Systems Manager Session Manager连接到实例，但您需要设置正确的基础设施。

### 工作原理

通常情况下，EC2实例需要互联网访问(通过IGW直接访问或通过NAT Gateway)才能与Systems Manager服务通信。但是，AWS提供了一种使用VPC端点(VPC Endpoints)在无互联网访问的情况下连接的方法。

### 所需设置

要在没有互联网访问的情况下启用Session Manager连接：

1. 配置以下服务的VPC端点：
   - com.amazonaws.[区域].ssm
   - com.amazonaws.[区域].ec2messages
   - com.amazonaws.[区域].ssmmessages
2. 确保这些VPC端点有适当的安全组规则
3. 验证IAM权限：
   - 实例必须有带有适当SSM权限的IAM角色
   - 尝试连接的用户需要Session Manager权限
4. 确保实例上已安装并运行SSM Agent

### 优势

这种方法提供：
- 增强的安全性(无互联网暴露)
- 在隔离的VPC环境中的连接能力
- 符合限制互联网访问的法规要求

这是一种常见的架构模式，适用于实例不应该有互联网连接但仍需要管理访问的安全环境。

![](https://raw.githubusercontent.com/liangyimingcom/storage/master/PicGo/20250328115950182.png)

![](https://raw.githubusercontent.com/liangyimingcom/storage/master/PicGo/20250328115950182.png)



## 2、AWS S3桶配置指南：仅限VPC内无公网出口子网访问

### 2.1 背景介绍

本文档详细描述了如何配置Amazon S3存储桶，使其仅允许通过VPC内部的无公网出口子网访问，同时通过HTTP/HTTPS接口提供服务。这种配置适用于对数据安全性要求较高的场景，确保数据只能在企业内部网络环境中访问。

### 2.2 架构概述

配置使用两种VPC端点类型：
- 网关端点：用于基本的S3访问
- 接口端点：提供HTTP/HTTPS接口访问能力

通过启用私有DNS，可以使用标准S3 URL直接在VPC内部访问S3资源，而无需修改应用程序代码。

### 2.3 实施步骤

#### 2.3.1 创建S3存储桶并关闭公共访问

1. 登录AWS管理控制台，导航至S3服务
2. 点击"创建存储桶"按钮
3. 输入存储桶名称（例如：s3httpaccesswithprivatesubnet02）
4. 选择适合的区域（例如：us-west-2）
5. 阻止所有公共访问：确保所有公共访问阻止选项都被勾选
6. 完成其余设置（加密、标签等）
7. 点击"创建存储桶"完成创建

#### 2.3.2 上传测试内容

1. 创建一个简单的index.html文件：

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
   <meta charset="UTF-8">
   <meta name="viewport" content="width=device-width, initial-scale=1.0">
   <title>Hello World</title>
   <style>
       body {
           font-family: 'Arial', sans-serif;
           display: flex;
           justify-content: center;
           align-items: center;
           height: 100vh;
           margin: 0;
           background-color: #f0f0f0;
      }
       .container {
           text-align: center;
           padding: 2rem;
           background-color: white;
           border-radius: 8px;
           box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
      }
       h1 {
           color: #333;
      }
       p {
           color: #666;
           margin-top: 1rem;
      }
   </style>
</head>
<body>
   <div class="container">
       <h1>Hello World</h1>
       <p>这是一个简单的网页示例</p>
   </div>
</body>
</html>
```

2. 上传此文件到S3存储桶

#### 2.3.3 创建VPC网关端点

1. 导航至VPC服务 → 端点
2. 点击"创建端点"按钮
3. 配置设置：
   - 服务类别：AWS服务
   - 服务名称：com.amazonaws.us-west-2.s3
   - VPC：选择您的VPC（例如：vpc-04fe37b08c6273a9e）
   - 路由表：选择包含无公网出口子网的路由表
4. 点击"创建端点"

#### 2.3.4 创建VPC接口端点

1. 导航至VPC服务 → 端点
2. 点击"创建端点"按钮
3. 配置设置：
   - 服务类别：AWS服务
   - 服务名称：com.amazonaws.us-west-2.s3（注意选择接口类型的服务）
   - VPC：选择您的VPC（例如：vpc-04fe37b08c6273a9e）
   - 子网：选择无公网出口的子网
   - 启用DNS名称：勾选此选项（重要！）
   - 安全组：选择或创建允许HTTPS流量的安全组
4. 点击"创建端点"
5. 记录端点ID（例如：vpce-0b0354085ac5093f4）

#### 2.3.5 通过ACL Enable公开桶，并且配置IP访问白名单（建议这么配容易）

#### 2.3.6 配置S3桶策略（可选跳过）

1. 返回S3服务，选择您的存储桶
2. 点击"权限"选项卡 → "存储桶策略" → "编辑"
3. 添加以下策略（替换相应的值）：

```json
{
 "Version": "2012-10-17",
 "Statement": [
    {
     "Sid": "DenyAccessExceptThroughVPCE",
     "Effect": "Deny",
     "Principal": "*",
     "Action": "s3:*",
     "Resource": [
       "arn:aws:s3:::s3httpaccesswithprivatesubnet02",
       "arn:aws:s3:::s3httpaccesswithprivatesubnet02/*"
    ],
     "Condition": {
       "StringNotEquals": {
         "aws:sourceVpce": "vpce-0b0354085ac5093f4"
      }
    }
    }
 ]
}
```

4. 点击"保存更改"

#### 2.3.7 配置IAM角色和策略（可选跳过）

1. 创建IAM策略允许访问S3存储桶：

```json
{
 "Version": "2012-10-17",
 "Statement": [
    {
     "Effect": "Allow",
     "Action": [
       "s3:ListBucket"
    ],
     "Resource": [
       "arn:aws:s3:::s3httpaccesswithprivatesubnet02"
    ]
    },
    {
     "Effect": "Allow",
     "Action": [
       "s3:GetObject",
       "s3:PutObject",
       "s3:DeleteObject"
    ],
     "Resource": [
       "arn:aws:s3:::s3httpaccesswithprivatesubnet02/*"
    ]
    }
 ]
}
```

2. 将此策略附加到需要访问S3的EC2实例角色

## 3、访问测试

### 3.1 基本访问测试

在VPC内的EC2实例上执行以下命令：

```bash
# 使用标准S3 URL（推荐方法）
curl https://s3httpaccesswithprivatesubnet02.s3.us-west-2.amazonaws.com/index.html

# 输出应该显示index.html的内容
```

### 3.2 接口端点URL测试

接口端点URL格式不是：
```
https://s3httpaccesswithprivatesubnet02.vpce-0b0354085ac5093f4-rp8g86rp.s3.us-west-2.vpce.amazonaws.com/index.html
```

而应该是：
```
https://vpce-0b0354085ac5093f4-rp8g86rp.s3.us-west-2.vpce.amazonaws.com/s3httpaccesswithprivatesubnet02/index.html
```

注意：由于启用了私有DNS，一般不需要使用接口端点特定的URL，使用标准S3 URL即可。

### 3.3 使用AWS CLI测试

```bash
# 列出桶中的对象
aws s3 ls s3://s3httpaccesswithprivatesubnet02/

# 下载对象
aws s3 cp s3://s3httpaccesswithprivatesubnet02/index.html ./downloaded.html
```

## 4、故障排除

### 4.1 SSL证书验证错误

如果遇到SSL证书验证错误，可以暂时使用-k选项以测试目的绕过：

```bash
curl -k https://vpce-endpoint-specific-dns.s3.region.vpce.amazonaws.com/bucket-name/key
```

注意：在生产环境中，应该解决证书问题而不是禁用验证。

### 4.2 访问被拒绝

检查以下几点：
- 确认IAM权限设置正确
- 验证桶策略中的VPC端点ID是否正确
- 检查是否从正确的VPC内访问

### 4.3 端点DNS问题

如果端点DNS解析有问题：
- 验证私有DNS是否已启用
- 确认EC2实例使用VPC提供的DNS服务器
- 检查DHCP选项集配置

## 5、最佳实践

1. 使用标准S3 URL: 由于启用了私有DNS，应用程序可以继续使用标准S3 URL，而无需修改代码。
2. 严格控制桶策略: 确保桶策略只允许通过指定VPC端点访问。
3. 定期审核安全设置: 定期检查IAM权限、桶策略和VPC端点配置。
4. 监控访问: 启用S3访问日志和CloudTrail，监控所有S3操作。
5. 考虑使用S3访问点: 对于更复杂的访问控制需求，可以考虑使用S3访问点。

## 6、结论

通过上述配置，我们成功实现了仅允许VPC内部无公网出口子网通过HTTP/HTTPS接口访问S3存储桶的目标。这种配置既保持了S3的易用性和兼容性，又显著提高了数据安全性，防止了未经授权的访问和数据泄露风险。

## 7、过程截图：成功

```bash
curl https://s3httpaccesswithprivatesubnet02.s3.us-west-2.amazonaws.com/index.html

curl -k https://vpce-0b0354085ac5093f4-rp8g86rp.s3.us-west-2.vpce.amazonaws.com/s3httpaccesswithprivatesubnet02/index.html
```



![](https://raw.githubusercontent.com/liangyimingcom/storage/master/PicGo/20250328122316237.png)

![](https://raw.githubusercontent.com/liangyimingcom/storage/master/PicGo/20250328122343107.png)

![](https://raw.githubusercontent.com/liangyimingcom/storage/master/PicGo/20250328122354367.png)
