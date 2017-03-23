# API 规范文档
[TOC]

## **简介**

## **适用范围**
	本文档适用于“深圳股利科技有限公司”API开发指导

## **URI规范**
1. URI 中的名词表示资源集合，使用复数形式;
2. 统一使用驼峰命名方法
3. 参数列表要encode；
4. URI设计遵循RESTful规范
4. 能直观通过URI的名字知道接口的功能

##### 版本
在URI中放版本信息：`GET /v1/users/1`

##### 避免层级过深的URI
层级不超过 2 层
过深的导航容易导致url膨胀，不易维护，如 `GET /zoos/1/areas/3/animals/4`，尽量使用查询参数代替路径中的实体导航，如 ` GET /animals?zoo=1&area=3`；

##### 对Composite资源的访问
###### 服务器端的组合实体必须在uri中通过父实体的id导航访问。
组合实体不是first-class的实体，它的生命周期完全依赖父实体，无法独立存在，在实现上通常是对数据库表中某些列的抽象，不直接对应表，也无id。一个常见的例子是 User — Address，Address是对User表中zipCode/country/city三个字段的简单抽象，无法独立于User存在。必须通过User索引到Address：GET /user/1/addresses

## **Request**
##### 请求头
Accept代表发送端（客户端）接受的数据类型，Accept 头的值必须是 application/json
##### 编码
所有请求和响应均 必须 使用 utf-8 为编码。
此标准体现在请求和响应的 Content-Type 头的值后必须配有 ;charset=utf-8 字样。
对于前端使用JSONP的场景，则应当为 `<script>` 标签添加 `charset="utf-8"` 属性。

##### 通过标准HTTP方法对资源CRUD
```
GET /zoos：列出所有动物园
POST /zoos：新建一个动物园
GET /zoos/ID：获取某个指定动物园的信息
PUT /zoos/ID：更新某个指定动物园的信息（提供该动物园的全部信息）
PATCH /zoos/ID：更新某个指定动物园的信息（提供该动物园的部分信息）
DELETE /zoos/ID：删除某个动物园
GET /zoos/ID/animals：列出某个指定动物园的所有动物
DELETE /zoos/ID/animals/ID：删除某个指定动物园的指定动物
```
##### 安全性和幂等性
1. 安全性：不会改变资源状态，可以理解为只读的
2. 幂等性：执行1次和执行N次，对资源状态改变的效果是等价的。

.   |  安全性   |   幂等性
----|----------|--------
GET |  √	     |   √
POST|  ×	     |   ×
PUT |  ×       |   √  
DELETE|   ×     |  √
安全性和幂等性均不保证反复请求能拿到相同的response。以 DELETE 为例，第一次DELETE返回200表示删除成功，第二次返回404提示资源不存在，这是允许的。

##### 复杂查询
查询可以捎带以下参数：

.   |  示例   |   备注
----|----------|--------
过滤条件| ?type=1&age=16	| 允许一定的uri冗余，如/zoos/1与/zoos?id=1
排序   | ?sort=age,desc	  |
投影	 | ?whitelist=id,name,email	|
分页   | ?page=10         |

## **Response**
##### response 的 body 直接就是数据，不要做多余的包装。
##### 各HTTP方法成功处理后的数据格式
.         | response 格式
----------|------------|
GET	      | 单个对象、集合
POST      | 新增成功的对象
PUT/PATCH	| 更新成功的对象
DELETE    | 空

##### json格式的约定
- 时间用长整形(毫秒数)，客户端自己按需解析（moment.js）
- 不传null字段

##### 成功模板
```
单个对象：
{
	'xxxx' : 'xxxxxx',
	'xxxx' : 'xxxxxx'
}
集合：
{
		'data' :
		{
			{
					'xxxx' : 'xxxxxx',
					'xxxx' : 'xxxxxx'
			},
			{
					'xxxx' : 'xxxxxx',
					'xxxx' : 'xxxxxx'
			}
		}
}
带分页的集合
{
		'data' :
		{
				{
					'xxxx' : 'xxxxxx',
					'xxxx' : 'xxxxxx'
				},
				{
					'xxxx' : 'xxxxxx',
					'xxxx' : 'xxxxxx'
			}
		}
		'meta':{
				'pagination':{
					"total": 2,
					"count": 2,
					"per_page": 10,
					"current_page": 1,
					"total_pages": 1,
					"links": []
				}
		}
}
```  

## **错误处理**
1. 不要发生了错误但给2xx响应，客户端可能会缓存成功的http请求；
2. 正确设置http状态码，不要自定义；
3. Response body 提供 1:错误的代码（日志/问题追查）。2:错误的描述文本（展示给用户）。

##### 错误模板
```
{
	code: 20001,
	message:"string"
	errors: ['xxx':['xxxxx'] ]
}
```

- `code` {integer} 错误码内容

- `message` {String}  错误信息

- `errors`  {Array}  表单验证错误的字段信息（验证规则通过不显示）

###### 抒写业务代码与文档同时，必须同时整理一份业务错误代码文档
+ 如错误码为20301时

2        |  03      |  01
---------|----------|----------
服务级错误（1为系统级错误）|服务模块代码|具体错误代码
##### 常用的http状态码及使用场景：
状态码 | 含义 | 使用场景
----|------|----
400 | BAD REQUEST  | 常用在参数校验
401 | UNAUTHORIZED  | 未经验证的用户，常见于未登录。如果经过验证后依然没权限，应该 403（即 authentication 和 authorization 的区别）。
403 | FORBIDDEN  | 无权限
404 | NOT FOUND  | 资源不存在
500 | INTERNAL SERVER ERROR  | 非业务类异常

###### 网络异常，不能正确完成请求流程，则客户端自行处理。客户端显示：
```
服务器异常，请稍后重试。
```

## **相关处理**
##### 空值和空字段处理
空值统一返回
根据不同类型返回不同的值，
数字为 0，字符串就是空字符串 ""，数组就是空数组 []，对象就是空对象 {}

##### 返回 URL
返回 URL 字段一律由后端返回完整 URL ，客户端直接使用该值即可
```
{
    url:"http://www.xxxxxx.com/app/view/1"
}
```

## **编写API文档模板规范**
开发人员编写 API 文档帮助说明时，必须至少包含如下几项：

1. API简介
	- 简单描述该 API
2. 请求
	- 请求该 API 的方法，地址
	- URI 参数： 如果 URI 某部分时动态的，请使用大括号说明：/api/user/{id}
	- URI 查询参数：如果 URI 地址有参数，描述各项参数的说明，每个参数是否可选
	- 请求标头：如果有特殊的请求标头，需要特别逐一说明
3. 响应结果
	- 说明响应的状态码、内容格式
	- 响应标头：如果有特殊的请求标头，需要特别逐一说明。
	- 响应正文：特殊字段、重点必须说明含义。尽量说明响应正文的所有字段意义。
		-	可选：授权、备注
		- 示例请求与响应

### **流程**
![流程图](http://ww1.sinaimg.cn/large/8c4739e3gw1f5dic3zc9sj20xy0hydkc.jpg)
