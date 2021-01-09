# TomorrowCampus
---
sidebar: auto
---

# 明日校园 API 文档

本项目前后端接口规范和接口文档。

[![Build Status](https://travis-ci.com/EyesCMS/zhishe-cms-doc.svg?branch=master)](https://travis-ci.com/EyesCMS/zhishe-cms-doc)

> 本文正在持续更新中~

## 1 前后端接口规范

### 1.1 请求格式

采用 RESTful 风格的接口

#### 1.1.1 HTTP 动词（HTTP verbs）

|   Verb   |        Description         |
| :------: | :------------------------: |
|  `HEAD`  |     获取 HTTP 头部信息     |
|  `GET`   |          获取资源          |
|  `POST`  |          创建资源          |
| `PATCH`  | 更新部分 JSON 内容，不常用 |
|  `PUT`   |          更新资源          |
| `DELETE` |          删除资源          |

#### 1.1.2 分页（Pagination）

API 自动对请求的元素分页，不同的 API 有不同的默认值，可以指定查询的最大长度，但对某些资源不起作用。**请求后端数组数据时，统一传递四个分页参数**。

**Parameters**

| 参数    | 含义                                |
| ------- | ----------------------------------- |
| `page`  | 请求页码                            |
| `limit` | 每页的元素数量                      |
| `sort`  | 排序字段，例如 "add_time" 或者 "id" |
| `order` | 升序降序，只能是 "desc" 或者 "asc"  |

这里四个参数是可选的，后端应该设置默认参数，因此即使前端不设置， 后端也会自动返回合适的对象数组响应数据。

Example

```
GET /goods/list?page=1&limit=10
```

Response

```http
HTTP/1.1 200 OK

{
  "totalCount": 2525,
  "items": [
  ...
  ]
}
```

**返回参数说明**

响应体里面 `totalCount` 表示总数量，`items` 是数组，里面是要查询的元素。

### 1.2 响应格式

```json
Content-Type: application/json;charset=UTF-8

{
  body
}
```

响应成功，body 直接返回 JSON 格式的内容（返回列表时按 1.1.2 分页）：

```json
{
  "age": 12,
  "name": "zhangsan"
}
```

#### 1.2.1 失败异常

错误响应，`errors` 字段可选（一般是参数验证错误）

```json
{
  "message": "Validation Failed",
  "errors": [
    {
      "resource": "Issue",
      "field": "title",
      "code": "missing_field"
    }
  ]
}
```

**客户端错误（Client errors）**

接收请求 body 的 API 调用上可能存在三种类型的客户端错误：

1. 发送无效的 JSON 会导致 `400 Bad Request` 的响应

   ```http
   HTTP/1.1 400 Bad Request
   Content-Length: 35

   {"message":"Problems parsing JSON"}
   ```

2. 发送错误类型的 JSON 值将导致 `400 Bad Request` 的响应

   ```http
   HTTP/1.1 400 Bad Request
   Content-Length: 40

   {"message":"Body should be a JSON object"}
   ```

3. 发送无效字段将导致 `422 Unprocessable Entity` 的响应

   ```http
   HTTP/1.1 422 Unprocessable Entity
   Content-Length: 149

   {
     "message": "Validation Failed",
     "errors": [
       {
         "resource": "Issue",
         "field": "title",
         "code": "missing_field"
       }
     ]
   }
   ```

   所有错误对象都具有资源和字段属性，以便你的客户端可以知道问题所在。还有一个错误代码，让你知道该字段出了什么问题。下面这些是可能的验证错误代码：

   |    Error Name    |          Description           |
   | :--------------: | :----------------------------: |
   |    `missing`     |           资源不存在           |
   | `missing_field`  |    资源上的必填字段尚未设置    |
   |    `invalid`     |          字段格式无效          |
   | `already_exists` | 另一个资源与此字段具有相同的值 |

   资源也可能发送自定义验证错误（其中 `code` 是自定义的）。自定义错误将始终具有一个描述错误的 `message` 字段，并且大多数错误还将包括指向一些可能有助于您解决该错误的内容的 `documentation_url`字段。

**登录失败**

- 认证失败

```http
HTTP/1.1 401 Unauthorized

{
  "message": "Full authentication is required to access this resource"
}
```

- 没有权限

```http
HTTP/1.1 403 Forbidden

{
  "message": "Access is denied"
}
```

### 1.3 状态码（Status code）

| code | message               | description                                                                      |
| ---- | --------------------- | -------------------------------------------------------------------------------- |
| 200  | OK                    | 请求成功                                                                         |
| 201  | Created               | 创建了新资源，通常是 POST 请求                                                   |
| 400  | Bad Request           | 由于语法无效，服务器无法理解请求（错误的 JSON 格式）                             |
| 401  | Unauthorized          | 未认证错误                                                                       |
| 403  | Forbidden             | 客户端没有访问内容的权限                                                         |
| 404  | Not Found             | 找不到请求的资源；服务器也可以发送此响应而不是 403，以隐藏来自未授权客户端的资源 |
| 405  | Method not Allowed    | 一般是请求方法使用错误，eg. `POST` vs `GET`                                      |
| 429  | Too Many Requests     | 在给定的时间内发送了太多请求(速率限制)                                           |
| 500  | Internal Server Error | 服务器遇到了不知道如何处理的情况                                                 |
| 503  | Service Unavailable   | 服务器尚未准备好处理请求，常见原因是服务器因维护而停机或过载                     |

参考 [MDC web docs](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status)

### 1.4 认证方式（Authentication）

前后端采用 token 来验证访问权限。

#### 1.4.1 Header & Token

前后端 Token 交换流程如下：

1. 前端访问系统登录 API
2. 成功以后，前端会接收后端响应的一个 token，保存在本地
3. 请求受保护 API ，则采用自定义头部携带此 token
4. 后端检验 Token，成功则返回受保护的数据

#### 1.4.2 自定义 Header

访问受保护 API 采用自定义 `Authorization` 头部

1. 前端访问后端登录 API`/auth/login`

   ```json
   POST /user/login

   {
       "username": "admin",
       "password": "123456"
   }
   ```

2. 成功以后，前端会接收后端响应的一个 token

   ```json
   {
     "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJ0aGlzIGlzIGxpdGVtYWxsIHRva2VuIiwiYXVkIjoiTUlOSUFQUCIsImlzcyI6IkxJVEVNQUxMIiwiZXhwIjoxNTU3MzI2ODUwLCJ1c2VySWQiOjEsImlhdCI6MTU1NzMxOTY1MH0.XP0TuhupV_ttQsCr1KTaPZVlTbVzVOcnq_K0kXdbri0",
     "tokenHead": "Bearer"
   }
   ```

3. 请求受保护 API 则，则采用自定义头部携带此 token

   ```json
   GET http://localhost:8080/users
   Authorization: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJ0aGlzIGlzIGxpdGVtYWxsIHRva2VuIiwiYXVkIjoiTUlOSUFQUCIsImlzcyI6IkxJVEVNQUxMIiwiZXhwIjoxNTU3MzM2ODU0LCJ1c2VySWQiOjIsImlhdCI6MTU1NzMyOTY1NH0.JY1-cqOnmi-CVjFohZMqK2iAdAH4O6CKj0Cqd5tMF3M
   ```


### 1.5 跨域资源共享（Cross origin resource sharing）

API 支持来自任何来源的 AJAX 请求的跨来源资源共享（CORS）

参考 [https://www.w3.org/TR/cors/](https://www.w3.org/TR/cors/)

## 2 用户服务 :hear_no_evil:

## 2.1 注册

用户注册接口

```
POST /user/register
```

#### Input

| Name       | Type     | Description |
| ---------- | -------- | ----------- |
| `username` | `string` | 用户名      |
| `password` | `string` | 密码        |

#### Example

```json
{
  "username": "yz141321",
  "password": "xxx",
}
```

#### Response

```json
{
    "code": 201,
    "message": "success",
    "data": "创建成功！"
}
```

### 2.2 登录

用户登录接口

```
POST /user/login
```

#### Input

| Name       | Type     | Description |
| :--------- | :------- | ----------- |
| `name`     | `string` | 用户名      |
| `password` | `string` | 密码        |


#### Example

```json
{
  "username": "yz141321",
  "password": "xxx",
}
```

#### Response

```json
{
    "code": 200,
    "message": "登录成功！",
    "data": {
        "userId": 3,
        "token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJhdWQiOiIzIn0.miLKM3eUVCEObVMNoQl3Ka2ago_36CwvQiIrQ9AL75E"
    }
}
```

**注意：**登陆失败返回 400，token 过期返回 401

### 2.3 拉取用户信息

使用 token 拉取用户详细信息

> token 放在请求头

```
GET /user/info
```

#### Response

```json
{
    "id": 3,
    "name": "yz",
    "grade": "2018",
    "major": "xxx",
    "profilePhoto": "333"
}
```


### 2.4 修改个人信息

用户修改个人信息

```
PUT /users/info
```

#### Input

| Name        | Type     | Description |
| ----------- | -------- | ----------- |
| `name`      | `string` | 昵称        |
| `grade`     | `string` | 年级        |
| `major`     | `string` | 专业        |


#### Example

```json
{
	  "name": "yz",
    "grade": "2018",
    "major": "xxx"
}
```

#### Response

```json
{
    "code": 200,
    "message": "success",
    "data": "修改成功"
}
```

### 2.5 退出登陆

用户退出登陆

```
POST /user/logout
```


## 3 帖子管理 :


### 3.1 发送帖子


```
POST /post//sendpost
```

#### Input

| Name        | Type     | Description |
| ----------- | -------- | ----------- |
| `name`      | `string` | 标题        |
| `content`   | `string` | 正文        |
| `type`      | `int`    | 类型        |

#### Example

```json
{
  "postid": 1,
  "content": "xxxxxxxxx",
  "type": 1
  
}
```

#### Response

```json
{
    "code": 201,
    "message": "success",
    "data": "发送成功！"
}
```

### 3.2 发送评论


```
POST /post/sendremark
```

#### Input

| Name        | Type     | Description |
| ----------- | -------- | ----------- |
| `postid`    | `int`    | 帖子ID      |
| `content`   | `string` | 正文        |

#### Example

```json
{
  "postid": 1,
  "content": "xxxxxxxxx"
  
}
```

#### Response

```json
{
    "code": 201,
    "message": "success",
    "data": "发送成功！"
}
```

### 3.4 查看全部帖子


```
GET /post/list
```

#### Response

```json
{
    "code": 200,
    "message": "success",
    "data": {
        "totalCount": 3,
        "items": [
            {
                "id": 1,
                "name": "text1",
                "content": "xxxxxxxxx",
                "type": 1,
                "date": "2021-01-08",
                "userId": 1
            },
            {
                "id": 2,
                "name": "text2",
                "content": "xxxxxxxxx",
                "type": 1,
                "date": "2021-01-08",
                "userId": 1
            },
            {
                "id": 4,
                "name": "yz test2",
                "content": "xxxxxxxxx",
                "type": 1,
                "date": "2021-01-09",
                "userId": 3
            }
        ]
    }
}
```

### 3.5 查看我的帖子


```
GET /post/mylist
```

#### Response

```json
{
    "code": 200,
    "message": "success",
    "data": {
        "totalCount": 1,
        "items": [
            {
                "id": 4,
                "name": "yz test2",
                "content": "xxxxxxxxx",
                "type": 1,
                "date": "2021-01-09",
                "userId": 3
            }
        ]
    }
}
```

### 3.6 查看具体帖子


```
GET /post/:postid/content
```

#### Response

```json
{
    "code": 200,
    "message": "success",
    "data": {
        "id": 4,
        "name": "yz test2",
        "content": "xxxxxxxxx",
        "type": 1,
        "date": "2021-01-09",
        "username": "yz",
        "photoList": [],
        "remarkList": [
            {
                "id": 4,
                "content": "yz给postid4的评论",
                "date": "2021-01-09",
                "username": "yz"
            }
        ]
    }
}
```



