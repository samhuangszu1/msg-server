# msg-server 企业微信会话存档服务 - 技术设计文档

## 目录
- [1. 项目概述](#1-项目概述)
- [2. 技术架构设计](#2-技术架构设计)
- [3. 核心模块详细设计](#3-核心模块详细设计)
- [4. API接口设计](#4-api接口设计)
- [5. 数据库设计](#5-数据库设计)
- [6. 部署与运维](#6-部署与运维)
- [7. 技术特色与创新点](#7-技术特色与创新点)
- [8. 安全与合规](#8-安全与合规)

## 1. 项目概述

### 1.1 项目定位
**项目背景**: msg-server是OpenSCRM项目中的**会话存档服务**模块，专门用于企业微信SCRM系统中消息的存储与管理。

**目标用户**: 企业微信SCRM系统管理员和开发者

**核心功能**:
- 企业微信会话消息的采集与存储
- 消息内容审计与合规管理
- 消息检索与导出
- 与企业微信官方SDK深度集成

### 1.2 业务价值
- **合规要求**: 满足企业消息存档的法规要求
- **数据安全**: 提供端到端的消息加密存储
- **业务洞察**: 支持消息内容分析和检索
- **系统集成**: 与企业微信生态无缝对接

## 2. 技术架构设计

### 2.1 整体架构模式
采用经典的**三层架构**模式：

```
┌─────────────────────────────────────────────┐
│                Controller Layer              │
│        (HTTP API层，处理路由和请求)           │
├─────────────────────────────────────────────┤
│                Service Layer                 │
│        (业务逻辑层，核心业务处理)             │
├─────────────────────────────────────────────┤
│                Model Layer                   │
│        (数据访问层，数据库操作)               │
└─────────────────────────────────────────────┘
```

### 2.2 核心技术栈

**语言与框架**:
- **Go 1.16+**: 主要开发语言，高性能并发处理
- **Gin**: 轻量级Web框架，处理HTTP路由
- **GORM**: ORM框架，负责数据库操作
- **Viper**: 配置管理，支持多种配置源

**数据存储**:
- **MySQL 8.0**: 主数据库，存储消息和会话数据
- **Redis**: 缓存层，用于延迟队列和会话管理

**外部依赖**:
- **企业微信官方SDK**: 通过CGO调用.so文件
- **阿里云OSS/腾讯云COS**: 文件存储服务
- **Snowflake**: 分布式ID生成

### 2.3 关键设计模式

**工厂模式**: 
- ID生成器（Snowflake）
- 存储策略（OSS、COS、本地）
- 日志组件

**策略模式**: 
- 存储后端切换（阿里云/腾讯云）
- 不同类型消息处理策略

**装饰器模式**: 
- 日志中间件
- 请求验证中间件

### 2.4 系统架构图

```
┌─────────────┐    ┌──────────────┐    ┌─────────────┐
│ 企业微信API  │────│ msg-server   │────│   MySQL     │
│   (SDK)     │    │    服务      │    │   数据库    │
└─────────────┘    └──────────────┘    └─────────────┘
                          │
                          ├─────────────────────────────
                          │                            │
                   ┌─────────────┐              ┌─────────────┐
                   │    Redis    │              │ 对象存储OSS │
                   │  延迟队列   │              │  文件存储   │
                   └─────────────┘              └─────────────┘
```

## 3. 核心模块详细设计

### 3.1 消息同步模块 (`services/msg_arch.go`)

#### 3.1.1 核心职责
负责从企业微信拉取消息并存储到本地数据库，是整个系统的数据入口。

#### 3.1.2 关键流程

```go
func (o *MsgArch) fetchAndStore(sdk *C.WeWorkFinanceSdk_t, priKeyPath string, extCorpID string) error {
    // 1. 读取RSA私钥，用于解密消息
    key, err := readPriKey(priKeyPath)
    
    // 2. 循环拉取消息
    for !noMoreMsg {
        // 获取上次同步的最大seq
        seq, _ := o.model.GetLatestSeq(extCorpID)
        
        // 调用企业微信SDK拉取消息
        chatDataSlice, _ := fetch(sdk, uint64(seq+1), batchSize, timeout, proxy, proxyPwd)
        
        // 3. 解密并处理每条消息
        for _, item := range chatData.ChatData {
            // 使用私钥解密随机密钥
            v15, _ := rsa.DecryptPKCS1v15(rand.Reader, key, encryptedKey)
            
            // 使用解密后的密钥解密消息内容
            C.DecryptData(randomKey, encryptedMsg, decryptSlice)
            
            // 4. 构建消息对象并生成会话ID
            msg.SessionID = o.getSessionID(msg, hash)
            msg.SessionType = o.getSessionType(msg)
            
            // 5. 下载附件（图片、文件等）
            if needDownload {
                mediaData, obj, _ := o.downloadFromWx(sdk, resp)
                storage.FileStorage.Put(obj, mediaData)
            }
        }
        
        // 6. 批量插入数据库
        o.model.Creat(chatMsgs)
    }
}
```

#### 3.1.3 技术亮点
- **增量同步**: 基于seq字段实现增量拉取，避免重复处理
- **加密解密**: RSA+AES双重加密保障数据安全
- **CGO集成**: 通过C语言调用企业微信官方SDK
- **批量处理**: 批量插入提高数据库写入性能

### 3.2 数据模型设计 (`models/chat_msg.go`)

#### 3.2.1 核心实体

**消息实体 (ChatMsg)**:
```go
type ChatMsg struct {
    ExtCorpModel        // 包含ID、企业ID、创建者ID
    MsgID      string   // 消息唯一标识
    Action     string   // 消息动作：send/recall/switch
    From       string   // 发送方ID
    ToList     []string // 接收方列表
    RoomID     string   // 群聊ID（单聊为空）
    MsgTime    int64    // 消息时间戳（毫秒）
    MsgType    string   // 消息类型：text/image/file等
    ContentText string  // 文本内容（支持全文检索）
    Seq        uint64   // 消息序号
    SessionID  string   // 会话ID（通过双方ID hash生成）
    SessionType string  // 会话类型：internal/external/group
    ChatMsgContent ChatMsgContent // 关联的附件内容
    Keywords   string   // 搜索关键词
}
```

#### 3.2.2 会话ID生成算法

```go
func (o *MsgArch) getSessionID(msg models.ChatMsg, h hash.Hash) string {
    if msg.RoomID != "" {
        // 群聊：发送方+群ID
        h.Write([]byte(msg.From + msg.RoomID))
    } else {
        // 单聊：按字典序排序双方ID
        if msg.From < msg.ToList[0] {
            h.Write([]byte(msg.From + msg.ToList[0]))
        } else {
            h.Write([]byte(msg.ToList[0] + msg.From))
        }
    }
    return hex.EncodeToString(h.Sum(nil))
}
```

**设计思路**:
- 确保同一会话的所有消息具有相同的SessionID
- 通过MD5 hash保证ID的唯一性和一致性
- 支持单聊和群聊两种场景

### 3.3 存储策略模块 (`common/storage/`)

#### 3.3.1 接口设计

```go
type FileStorageInterface interface {
    SignURL(objectKey string, method HTTPMethod, expiredInSec int64) (string, error)
    Get(objectKey string) (io.ReadCloser, error)
    Put(objectKey string, reader io.Reader) error
    IsExist(objectKey string) (bool, error)
    Delete(objectKeys ...string) ([]string, error)
}
```

#### 3.3.2 支持的存储后端

**阿里云OSS**:
- 高可用对象存储服务
- 支持CDN加速
- 成本效益高

**腾讯云COS**:
- 与企业微信生态更好集成
- 数据就近存储
- 安全合规

**动态配置切换**:
```yaml
Storage:
  Type: aliyun  # 或 qcloud、local
  AccessKeyId: xxx
  AccessKeySecret: xxx
  Endpoint: xxx
  Bucket: xxx
```

### 3.4 延迟队列模块 (`common/delay_queue/`)

#### 3.4.1 设计理念
基于Redis实现的分布式延迟队列，用于处理异步任务和定时任务。

#### 3.4.2 核心组件

- **Bucket**: 时间分片桶，存储待执行任务
- **ReadyQueue**: 就绪队列，存储可立即执行的任务
- **Timer**: 定时器，扫描bucket并转移到ReadyQueue

#### 3.4.3 算法流程

**添加任务**:
```go
func Add(job Job) error {
    // 存储任务详情到Redis Hash
    putJob(job.ID, job)
    // 根据执行时间放入对应bucket
    pushToBucket(bucketName, job.ExecuteAt, job.ID)
}
```

**定时扫描**:
```go
func tickHandler(t time.Time, bucketName string) {
    for {
        bucketItem := getFromBucket(bucketName)
        if bucketItem.timestamp <= t.Unix() {
            // 时间到了，转移到就绪队列
            pushToReadyQueue(job.Topic, bucketItem.jobID)
            removeFromBucket(bucketName, bucketItem.jobID)
        }
    }
}
```

## 4. API接口设计

### 4.1 REST API规范

**基础路径**: `/api/v1`

**核心接口**:
```go
apiV1.POST("/sync", msgArch.Sync)                    // 同步消息
apiV1.GET("/sessions", msgArch.QuerySessions)        // 查询会话列表
apiV1.GET("/msgs", msgArch.QueryChatMsgs)           // 查询消息记录
apiV1.GET("/search", msgArch.SearchMsgs)            // 搜索消息内容
```

### 4.2 安全认证机制

**HMAC签名验证**:
```go
func CheckMAC(message []byte, messageMAC string, key []byte) bool {
    mac := hmac.New(sha256.New, key)
    mac.Write(message)
    expectedMAC := mac.Sum(nil)
    return hex.EncodeToString(expectedMAC) == messageMAC
}
```

**JWT令牌认证**:
```go
type Claims struct {
    UID  string `json:"uid"`   // 用户ID
    Role string `json:"role"`  // 用户角色
    jwt.StandardClaims
}
```

## 5. 数据库设计

### 5.1 表结构设计

#### 5.1.1 消息表 (chat_msg)

```sql
CREATE TABLE chat_msg (
    id VARCHAR(20) PRIMARY KEY COMMENT '主键ID',
    ext_corp_id CHAR(18) NOT NULL COMMENT '企业ID',
    ext_creator_id CHAR(32) COMMENT '创建者ID',
    msg_id CHAR(128) UNIQUE COMMENT '消息唯一标识',
    action CHAR(8) COMMENT '消息动作',
    `from` CHAR(32) COMMENT '发送方ID',
    to_list JSON COMMENT '接收方列表',
    room_id CHAR(128) COMMENT '群聊ID',
    msg_time BIGINT COMMENT '消息时间戳',
    msg_type VARCHAR(32) COMMENT '消息类型',
    content_text VARCHAR(512) COMMENT '文本内容',
    seq BIGINT UNSIGNED COMMENT '消息序号',
    session_id VARCHAR(32) COMMENT '会话ID',
    session_type VARCHAR(32) COMMENT '会话类型',
    keywords VARCHAR(255) COMMENT '搜索关键词',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX idx_ext_corp_id (ext_corp_id),
    INDEX idx_session_id (session_id),
    INDEX idx_msg_time (msg_time),
    INDEX idx_from_to (ext_corp_id, `from`, session_id),
    FULLTEXT idx_content_text (content_text) WITH PARSER ngram
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 5.2 索引优化策略

**复合索引设计**:
```sql
-- 会话查询索引
CREATE INDEX idx_session_query ON chat_msg 
(ext_corp_id, session_type, session_id, msg_time DESC);

-- 全文检索索引（支持中文）
CREATE FULLTEXT INDEX idx_content_search ON chat_msg (content_text) 
WITH PARSER ngram;
```

## 6. 部署与运维

### 6.1 Docker化部署

#### 6.1.1 Dockerfile

```dockerfile
FROM golang:latest
WORKDIR /data/xjyk
ADD . .
ENV GOPROXY="https://goproxy.cn"
ENV CGO_ENABLED=1
RUN go mod download

# 设置企业微信SDK库路径
ENV LD_LIBRARY_PATH=${LD_LIBRARY_PATH}:/data/xjyk/lib

# 编译应用
RUN go build -ldflags="-s -w" -installsuffix cgo -o msg-arch .

EXPOSE 9002
CMD ["./msg-arch"]
```

#### 6.1.2 环境配置

**关键环境变量**:
```bash
# 企业微信SDK库路径
export LD_LIBRARY_PATH=$(pwd)/lib

# Go编译环境
export CGO_ENABLED=1
export GOPROXY=https://proxy.golang.com.cn,direct
```

### 6.2 配置管理

**生产环境配置**:
```yaml
App:
  Env: PROD
  Key: ${APP_SECRET_KEY}
  AutoMigration: false

Server:
  RunMode: release
  MsgArchHttpPort: 9002

DB:
  Host: ${DB_HOST}
  User: ${DB_USER}
  Password: ${DB_PASSWORD}
  Name: ${DB_NAME}
```

## 7. 技术特色与创新点

### 7.1 企业微信SDK深度集成
- **CGO调用**: 直接调用官方C/C++ SDK，性能更优
- **加密通信**: 支持RSA+AES双重加密
- **增量同步**: 基于seq机制的高效同步

### 7.2 高性能数据处理
- **批量写入**: 使用GORM的CreateInBatches优化
- **分页查询**: 支持游标分页和传统分页
- **全文检索**: MySQL ngram全文索引支持中文

### 7.3 灵活的存储策略
- **多云支持**: 阿里云OSS、腾讯云COS
- **策略模式**: 运行时动态切换存储后端
- **CDN集成**: 支持CDN加速文件访问

### 7.4 分布式延迟队列
- **Redis实现**: 基于Redis的高可用队列
- **时间分片**: Bucket机制提高扫描效率
- **水平扩展**: 支持多实例部署

## 8. 安全与合规

### 8.1 数据安全
- **端到端加密**: 消息在传输和存储中均加密
- **访问控制**: 基于JWT的身份认证
- **签名验证**: HMAC-SHA256防篡改

### 8.2 合规要求
- **数据审计**: 完整的消息存档记录
- **权限隔离**: 基于企业ID的数据隔离
- **备份恢复**: 支持数据备份和恢复

---

*本文档详细描述了msg-server项目的技术架构、核心模块设计、API接口、数据库设计等关键技术要点，为开发和维护提供重要参考。*