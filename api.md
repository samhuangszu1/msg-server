# msg-server API接口文档

## 概述

msg-server是企业微信会话存档服务，提供消息同步、会话查询、消息查询和搜索等功能的REST API接口。

### 基础信息
- **服务地址**: `http://localhost:9002`
- **API版本**: `v1`
- **基础路径**: `/api/v1`
- **数据格式**: JSON
- **字符编码**: UTF-8

### 认证方式
API使用HMAC-SHA256签名验证，请求需要携带签名参数。

### 通用响应格式
```json
{
    "code": 0,
    "message": "success",
    "data": {}
}
```

**响应状态码说明**:
- `0`: 成功
- `400`: 请求参数错误
- `401`: 认证失败
- `403`: 权限不足
- `500`: 服务器内部错误

## 接口列表

### 1. 消息同步接口

#### 1.1 触发消息同步

**接口描述**: 触发企业微信消息同步任务

**请求信息**:
- **URL**: `/api/v1/chat-msg/sync`
- **Method**: `POST`
- **Content-Type**: `application/json`

**请求参数**:
```json
{
    "ext_corp_id": "企业ID",
    "signature": "HMAC签名"
}
```

**参数说明**:
| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| ext_corp_id | string | 是 | 企业微信企业ID |
| signature | string | 是 | HMAC-SHA256签名 |

**响应示例**:
```json
{
    "code": 0,
    "message": "同步任务已启动",
    "data": null
}
```

**错误响应**:
```json
{
    "code": 400,
    "message": "签名验证失败",
    "data": null
}
```

### 2. 会话查询接口

#### 2.1 查询会话列表

**接口描述**: 查询用户的会话列表，支持分页和按会话类型筛选

**请求信息**:
- **URL**: `/api/v1/chat-msg/sessions`
- **Method**: `GET`

**请求参数**:
| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| ext_staff_id | string | 是 | 员工ID |
| session_type | string | 是 | 会话类型：internal(内部)/external(外部)/group(群聊) |
| ext_corp_id | string | 是 | 企业ID |
| name | string | 否 | 搜索会话对方姓名（模糊匹配） |
| page | int | 否 | 页码，默认1 |
| page_size | int | 否 | 每页大小，默认20，最大100 |
| signature | string | 是 | HMAC-SHA256签名 |

**请求示例**:
```
GET /api/v1/chat-msg/sessions?ext_staff_id=staff123&session_type=external&ext_corp_id=corp123&page=1&page_size=20&signature=abc123
```

**响应示例**:
```json
{
    "code": 0,
    "message": "success",
    "data": {
        "items": [
            {
                "id": "session001",
                "ext_corp_id": "corp123",
                "msgid": "msg123456",
                "action": "send",
                "from": "staff123",
                "tolist": ["customer456"],
                "room_id": "",
                "msg_time": 1640995200000,
                "msg_type": "text",
                "content_text": "您好，有什么可以帮助您的吗？",
                "seq": 12345,
                "session_id": "sess_abc123",
                "session_type": "external",
                "peer_avatar": "https://example.com/avatar.jpg",
                "peer_name": "张三",
                "peer_ext_id": "customer456",
                "peer_type": "1",
                "group_chat_name": ""
            }
        ],
        "total": 50,
        "page": 1,
        "page_size": 20
    }
}
```

**字段说明**:
| 字段名 | 类型 | 说明 |
|--------|------|------|
| items | array | 会话列表 |
| total | int | 总记录数 |
| page | int | 当前页码 |
| page_size | int | 每页大小 |
| peer_avatar | string | 会话对方头像URL |
| peer_name | string | 会话对方姓名 |
| peer_ext_id | string | 会话对方外部ID |
| peer_type | string | 对方类型：1-微信，2-企业微信 |
| msg_time | int64 | 消息时间戳（毫秒） |

### 3. 消息查询接口

#### 3.1 查询聊天消息

**接口描述**: 查询指定会话的聊天消息记录，支持时间范围筛选和分页

**请求信息**:
- **URL**: `/api/v1/chat-msg/session-msgs`
- **Method**: `GET`

**请求参数**:
| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| ext_staff_id | string | 是 | 员工ID |
| receiver_id | string | 是 | 接收方ID |
| ext_corp_id | string | 是 | 企业ID |
| msg_type | string | 否 | 消息类型：text/image/file/voice等 |
| send_at_start | string | 否 | 开始时间，格式：2006-01-02 |
| send_at_end | string | 否 | 结束时间，格式：2006-01-02 |
| max_id | string | 否 | 查询ID大于该值的消息（向后翻页） |
| min_id | string | 否 | 查询ID小于该值的消息（向前翻页） |
| page | int | 否 | 页码，默认1 |
| page_size | int | 否 | 每页大小，默认20 |
| sort_field | string | 否 | 排序字段，默认msg_time |
| sort_type | string | 否 | 排序方式：asc/desc，默认desc |
| signature | string | 是 | HMAC-SHA256签名 |

**请求示例**:
```
GET /api/v1/chat-msg/session-msgs?ext_staff_id=staff123&receiver_id=customer456&ext_corp_id=corp123&send_at_start=2024-01-01&send_at_end=2024-01-31&page=1&page_size=20&signature=abc123
```

**响应示例**:
```json
{
    "code": 0,
    "message": "success",
    "data": {
        "items": [
            {
                "id": "msg001",
                "ext_corp_id": "corp123",
                "msgid": "msg123456",
                "action": "send",
                "from": "staff123",
                "tolist": ["customer456"],
                "room_id": "",
                "msg_time": 1640995200000,
                "msg_type": "text",
                "content_text": "您好，有什么可以帮助您的吗？",
                "seq": 12345,
                "session_id": "sess_abc123",
                "session_type": "external",
                "sender_name": "客服小王",
                "sender_avatar": "https://example.com/staff_avatar.jpg",
                "chat_msg_content": {
                    "id": "content001",
                    "chat_msg_id": "msg001",
                    "content_type": "text",
                    "content": "{\"content\":\"您好，有什么可以帮助您的吗？\"}",
                    "file_url": "",
                    "file_name": ""
                },
                "attachments": null
            },
            {
                "id": "msg002",
                "ext_corp_id": "corp123",
                "msgid": "msg123457",
                "action": "send",
                "from": "customer456",
                "tolist": ["staff123"],
                "room_id": "",
                "msg_time": 1640995260000,
                "msg_type": "image",
                "content_text": "",
                "seq": 12346,
                "session_id": "sess_abc123",
                "session_type": "external",
                "sender_name": "张三",
                "sender_avatar": "https://example.com/customer_avatar.jpg",
                "chat_msg_content": {
                    "id": "content002",
                    "chat_msg_id": "msg002",
                    "content_type": "image",
                    "content": "{\"md5sum\":\"abc123\",\"sdkfileid\":\"file123\",\"filesize\":1024}",
                    "file_url": "https://example.com/images/file123.jpg",
                    "file_name": "screenshot.jpg"
                },
                "attachments": {
                    "md5sum": "abc123",
                    "sdkfileid": "file123",
                    "filesize": 1024
                }
            }
        ],
        "total": 100
    }
}
```

**消息类型说明**:
| 消息类型 | 说明 | attachments字段内容 |
|----------|------|-------------------|
| text | 文本消息 | null |
| image | 图片消息 | {md5sum, sdkfileid, filesize} |
| file | 文件消息 | {md5sum, filename, fileext, filesize} |
| voice | 语音消息 | {md5sum, voice_size, play_length, sdkfileid} |
| card | 名片消息 | {corpname, userid} |
| location | 位置消息 | {longitude, latitude, address, title, zoom} |
| emotion | 表情消息 | {type, width, height, imagesize, md5sum, sdkfileid} |
| link | 链接消息 | {title, description, link_url, image_url} |
| revoke | 撤回消息 | {pre_msgid} |

### 4. 消息搜索接口

#### 4.1 全文搜索消息

**接口描述**: 在指定会话中搜索包含关键词的消息

**请求信息**:
- **URL**: `/api/v1/chat-msg/search`
- **Method**: `GET`

**请求参数**:
| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| keyword | string | 是 | 搜索关键词 |
| ext_staff_id | string | 是 | 员工ID |
| ext_peer_id | string | 是 | 对方ID |
| ext_corp_id | string | 是 | 企业ID |
| page | int | 否 | 页码，默认1 |
| page_size | int | 否 | 每页大小，默认20 |
| signature | string | 是 | HMAC-SHA256签名 |

**请求示例**:
```
GET /api/v1/chat-msg/search?keyword=产品介绍&ext_staff_id=staff123&ext_peer_id=customer456&ext_corp_id=corp123&page=1&page_size=20&signature=abc123
```

**响应示例**:
```json
{
    "code": 0,
    "message": "success",
    "data": {
        "items": [
            {
                "id": "msg003",
                "ext_corp_id": "corp123",
                "msgid": "msg123458",
                "action": "send",
                "from": "staff123",
                "tolist": ["customer456"],
                "room_id": "",
                "msg_time": 1640995320000,
                "msg_type": "text",
                "content_text": "这是我们最新的产品介绍，请您查看",
                "seq": 12347,
                "session_id": "sess_abc123",
                "session_type": "external",
                "sender_name": "客服小王",
                "sender_avatar": "https://example.com/staff_avatar.jpg",
                "chat_msg_content": {
                    "id": "content003",
                    "chat_msg_id": "msg003",
                    "content_type": "text",
                    "content": "{\"content\":\"这是我们最新的产品介绍，请您查看\"}",
                    "file_url": "",
                    "file_name": ""
                },
                "attachments": null
            }
        ],
        "total": 5
    }
}
```

## 签名验证

### 签名生成规则

1. **参数排序**: 将所有请求参数（除signature外）按参数名进行字典序排序
2. **拼接字符串**: 将排序后的参数按 `key=value&key=value` 格式拼接
3. **生成签名**: 使用HMAC-SHA256算法，以配置的密钥对拼接字符串进行签名
4. **编码**: 将签名结果转换为十六进制字符串

### 签名示例

**请求参数**:
```
ext_staff_id=staff123
session_type=external
ext_corp_id=corp123
page=1
page_size=20
```

**排序后拼接**:
```
ext_corp_id=corp123&ext_staff_id=staff123&page=1&page_size=20&session_type=external
```

**生成签名** (假设密钥为 `secret_key`):
```javascript
const crypto = require('crypto');
const data = 'ext_corp_id=corp123&ext_staff_id=staff123&page=1&page_size=20&session_type=external';
const key = 'secret_key';
const signature = crypto.createHmac('sha256', key).update(data).digest('hex');
```

## 错误码说明

| 错误码 | 说明 |
|--------|------|
| 0 | 成功 |
| 400 | 请求参数错误 |
| 401 | 签名验证失败 |
| 403 | 权限不足 |
| 404 | 资源不存在 |
| 500 | 服务器内部错误 |
| 1001 | 企业ID无效 |
| 1002 | 员工ID无效 |
| 1003 | 会话类型无效 |
| 1004 | 消息类型无效 |
| 1005 | 时间格式错误 |
| 1006 | 分页参数错误 |

## 接口调用示例

### JavaScript示例

```javascript
// 生成签名
function generateSignature(params, secret) {
    const crypto = require('crypto');
    
    // 排序参数
    const sortedParams = Object.keys(params).sort().reduce((result, key) => {
        result[key] = params[key];
        return result;
    }, {});
    
    // 拼接字符串
    const queryString = Object.keys(sortedParams)
        .map(key => `${key}=${sortedParams[key]}`)
        .join('&');
    
    // 生成签名
    return crypto.createHmac('sha256', secret).update(queryString).digest('hex');
}

// 查询会话列表
async function getSessionList() {
    const params = {
        ext_staff_id: 'staff123',
        session_type: 'external',
        ext_corp_id: 'corp123',
        page: 1,
        page_size: 20
    };
    
    const secret = 'your_secret_key';
    const signature = generateSignature(params, secret);
    
    const url = new URL('/api/v1/chat-msg/sessions', 'http://localhost:9002');
    Object.keys(params).forEach(key => url.searchParams.append(key, params[key]));
    url.searchParams.append('signature', signature);
    
    const response = await fetch(url);
    const result = await response.json();
    
    console.log(result);
}
```

### Python示例

```python
import hmac
import hashlib
import requests
from urllib.parse import urlencode

def generate_signature(params, secret):
    """生成签名"""
    # 排序参数
    sorted_params = sorted(params.items())
    
    # 拼接字符串
    query_string = urlencode(sorted_params)
    
    # 生成签名
    signature = hmac.new(
        secret.encode('utf-8'),
        query_string.encode('utf-8'),
        hashlib.sha256
    ).hexdigest()
    
    return signature

def get_chat_messages():
    """查询聊天消息"""
    params = {
        'ext_staff_id': 'staff123',
        'receiver_id': 'customer456',
        'ext_corp_id': 'corp123',
        'page': 1,
        'page_size': 20
    }
    
    secret = 'your_secret_key'
    signature = generate_signature(params, secret)
    params['signature'] = signature
    
    url = 'http://localhost:9002/api/v1/chat-msg/session-msgs'
    response = requests.get(url, params=params)
    result = response.json()
    
    print(result)
```

## 注意事项

1. **时区处理**: 所有时间戳均为UTC时间的毫秒级时间戳
2. **文件访问**: 文件URL为带签名的临时访问链接，有效期为1年
3. **分页限制**: 单次查询最大返回100条记录
4. **频率限制**: API调用频率限制为每秒100次请求
5. **字符编码**: 所有文本内容使用UTF-8编码
6. **JSON格式**: content字段中的JSON内容需要进行转义处理

## 更新记录

| 版本 | 日期 | 更新内容 |
|------|------|----------|
| v1.0 | 2024-01-01 | 初始版本，包含基础的消息同步、会话查询、消息查询和搜索功能 |

---

如有疑问，请联系技术支持团队。