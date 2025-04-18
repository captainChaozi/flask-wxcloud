开发指引 /调用云托管服务 /微信小程序
微信小程序-访问云托管服务
微信小程序中操作微信云托管服务，使用小程序基础库中 wx.cloud 方法对象，无需引入额外的 SDK。

需要注意，微信小程序基础库版本应该在 2.23.0 及以上，请务必配套在「小程序管理后台」-「设置」-「功能设置」-「基础库最低版本设置」中，将值设定为 2.23.0 及以上。

使用优势(对比 wx.request)
不耗费任何公网流量，前后端通信走内网；
天然免疫 DDoS 攻击，仅授权小程序/公众号可访问后端，其他人即便拿到环境 id 和服务名也无法访问；
通过微信就近接入节点加速，无视后端服务地域影响，没有跨地域延迟，后端无需多地部署；
无需在小程序后台配置「服务器域名」；
后端可直接获取用户信息，无需调接口即可以获取 opneid 等。
因此，如果云托管服务只有微信小程序/公众号会调用，建议在服务设置中关闭公网访问。

使用前提
最简单的情况，「小程序」对「直属微信云托管环境」的服务进行访问，也就是登录微信云托管控制台时选择小程序进入的环境。

以上情况我们约定为 A 小程序，如果你希望另外一个 B 小程序/B 公众号 也可以访问 A 小程序 的云托管环境：

A 小程序 和 B 小程序/B 公众号同主体：需要配置资源复用。

A 小程序 和 B 小程序/B 公众号不同主体：推荐使用「微信开放平台-第三方平台」方式。

A 小程序和 B 小程序/B 公众号不同主体：将 B 小程序/B 公众号视为「其他客户端」，通过公网访问。参考文档其他客户端-访问云托管服务。此方式下，B 小程序/B 公众号无法使用云调用/微信令牌，且需要配置「服务器域名」（使用云托管服务的默认公网域名/自定义域名均可）。

如果你对以上正在支持的能力有任何建议和期待，欢迎在官方交流群联系我们。

基本使用
在小程序中使用如下的代码（取代原有 wx.request 用法）：

// 确认已经在 onLaunch 中调用过 wx.cloud.init 初始化环境（任意环境均可，可以填空）
const res = await wx.cloud.callContainer({
config: {
env: '填入云环境 ID', // 微信云托管的环境 ID
},
path: '/xxx', // 填入业务自定义路径和参数，根目录，就是 /
method: 'POST', // 按照自己的业务开发，选择对应的方法
header: {
'X-WX-SERVICE': 'xxx', // xxx 中填入服务名称（微信云托管 - 服务管理 - 服务列表 - 服务名称）
// 其他 header 参数
}
// dataType:'text', // 默认不填是以 JSON 形式解析返回结果，若不想让 SDK 自己解析，可以填 text
// 其余参数同 wx.request
});

console.log(res);
如上使用前，需要在小程序 app.js 中，执行 wx.init，如下代码：

App({
async onLaunch() {
// 使用 callContainer 前一定要 init 一下，全局执行一次即可
wx.cloud.init()
// 下面的请求可以在页面任意一处使用
const result = await wx.cloud.callContainer({
config: {
env: 'prod-01', // 微信云托管的环境 ID
},
path: '/', // 填入业务自定义路径和参数，根目录，就是 /
method: 'GET', // 按照自己的业务开发，选择对应的方法
header: {
'X-WX-SERVICE': 'xxx', // xxx 中填入服务名称（微信云托管 - 服务管理 - 服务列表 - 服务名称）
}
// dataType:'text', // 默认不填是以 JSON 形式解析返回结果，若不想让 SDK 自己解析，可以填 text
})
console.log(result)
}
})
如果资源复用情况下，需要在 wx.cloud.init 中填写环境来源的账号 appid，

例如，小程序 A（appid：WXAAA）的环境 prod_001 授权给 小程序 B（appid：WXBBB），则应是如下：

const c1 = new wx.cloud.Cloud({
resourceAppid: 'WXAAA', // 环境所属的账号 appid
resourceEnv: 'prod_001', // 微信云托管的环境 ID
})
await c1.init()
await c1.callContainer({
config: {
env: 'prod_001', // 微信云托管的环境 ID
},
path: '/xxx', // 填入业务自定义路径和参数，根目录，就是 /
method: 'POST', // 按照自己的业务开发，选择对应的方法
// dataType:'text', // 如果返回的不是 json 格式，需要添加此项
header: {
'X-WX-SERVICE': 'xxx', // xxx 中填入服务名称（微信云托管 - 服务管理 - 服务列表 - 服务名称）
// 其他 header 参数
}
// 其余参数同 wx.request
})
如果你想深入了解 callContainer 的原理，建议阅读这篇文章

在这里我们提供了一个万能的封装方法，无论是自己的小程序还是资源复用形态，均可以正常使用。

在小程序 app.js 中建立一个 call 方法，并将 appid、环境 ID、服务名称填入适当的位置

App({
async onLaunch() {
// 这里演示如何在 app.js 中使用，没有用请删掉
// const res = await this.call({
// path:'/',
// method: 'POST'
// })
// console.log('业务返回结果',res)
},
/\*\*

- 封装的微信云托管调用方法
- @param {\*} obj 业务请求信息，可按照需要扩展
- @param {_} number 请求等待，默认不用传，用于初始化等待
  _/
  async call(obj, number=0){
  const that = this
  if(that.cloud == null){
  const cloud = new wx.cloud.Cloud({
  resourceAppid: 'WXAAA', // 微信云托管环境所属账号，服务商 appid、公众号或小程序 appid
  resourceEnv: 'prod-001', // 微信云托管的环境 ID
  })
  that.cloud = cloud
  await that.cloud.init() // init 过程是异步的，需要等待 init 完成才可以发起调用
  }
  try{
  const result = await that.cloud.callContainer({
  path: obj.path, // 填入业务自定义路径和参数，根目录，就是 /
  method: obj.method||'GET', // 按照自己的业务开发，选择对应的方法
  // dataType:'text', // 如果返回的不是 json 格式，需要添加此项
  header: {
  'X-WX-SERVICE': 'xxx', // xxx 中填入服务名称（微信云托管 - 服务管理 - 服务列表 - 服务名称）
  // 其他 header 参数
  }
  // 其余参数同 wx.request
  })
  console.log(`微信云托管调用结果${result.errMsg} | callid:${result.callID}`)
  return result.data // 业务数据在 data 中
  } catch(e){
  const error = e.toString()
  // 如果错误信息为未初始化，则等待 300ms 再次尝试，因为 init 过程是异步的
  if(error.indexOf("Cloud API isn't enabled")!=-1 && number<3){
  return new Promise((resolve)=>{
  setTimeout(function(){
  resolve(that.call(obj,number+1))
  },300)
  })
  } else {
  throw new Error(`微信云托管调用失败${error}`)
  }
  }
  }
  })
  在 page 页面 js 中，可以如下使用：

const app = getApp()
Page({
async onLoad(){
const res = await app.call({
path:'/'
})
console.log('业务返回结果',res)
}
})

请求参数
wx.cloud.callContainer 其他参数，直接参考 wx.request API，在这里列举常用参数：

属性 类型 默认值 必填 说明 最低版本
config.env string 是 微信云托管环境 ID
path string 是 后端服务接口地址
data string/object/ArrayBuffer 否 请求的参数
header Object 是 设置请求的 header，header 中不能设置 Referer。
content-type 默认为 application/json
timeout number 否 超时时间，单位为毫秒.最大值不能超过 15 秒，否则无效
method string GET 否 HTTP 请求方法
dataType string json 否 返回的数据格式
responseType string text 否 响应的数据类型
success function 否 接口调用成功的回调函数
fail function 否 接口调用失败的回调函数
complete function 否 接口调用结束的回调函数（调用成功、失败都会执行）
如果希望 wx.cloud.container 返回 Promise，请勿传 success, fail 和 complete

object.method 的合法值
值 说明 最低版本
OPTIONS HTTP 请求 OPTIONS
GET HTTP 请求 GET
HEAD HTTP 请求 HEAD
POST HTTP 请求 POST
PUT HTTP 请求 PUT
DELETE HTTP 请求 DELETE
TRACE HTTP 请求 TRACE
CONNECT HTTP 请求 CONNECT
object.dataType 的合法值
值 说明 最低版本
json 返回的数据为 JSON，返回后会对返回的数据进行一次 JSON.parse
其他 不对返回的内容进行 JSON.parse
object.responseType 的合法值
值 说明 最低版本
text 响应的数据为文本
arraybuffer 响应的数据为 ArrayBuffer
后端直接获取用户信息
小程序向云托管服务发起 callcontainer 调用时，你的服务请求 header 中会自动带有用户信息，包括 openid、unionid、ip 地址、可信来源等等，无需再通过小程序 wx.login 登录，然后调接口置换，大幅简化了流程。

注意，资源复用情况下，获取 openid 的字段和普通获取不一致。

使用限制
请求大小限制 100KiB (对象类型限制 20 MiB，请求中不建议包含图片，可通过对象存储处理)；
返回包大小限制 1000KiB。

push
