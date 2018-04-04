# 基于阿里云函数计算的云相册搭建指南
## 目标
我们的终极目标是搭建一个私人云相册，相册的拥有者可以上传图片，生成相册，相册拥有者的小伙伴通过用户名密码登录，访问相册，查看图片。
效果请访问[Photo-Gallery Based on FC Compute](http://photo-gallery-cici.oss-cn-shanghai.aliyuncs.com/login/index.html)，用户名`username`，密码`password`

## 方案设计
我想构建一个私人云相册，但却不想关心运行、维护服务器等底层琐碎的事情，那用[阿里云函数计算](https://www.aliyun.com/product/fc?spm=5176.173847.765261.442.vJgQAV)再合适不过了。函数计算使用户无需管理服务器等基础设施，只需编写代码并上传就可以快速构建任何类型的应用和服务，无需管理和运维。函数计算会为用户准备好计算资源，以弹性、可靠的方式运行代码，并提供日志查询，性能监控，报警等功能。

博主对比了函数计算和传统方式搭建服务的差异，如图所示。
| Item     |   函数计算FC | 传统方式搭建服务  |
| :--------: | :--------: | :--: |
| 维护成本  | 维护成本低，无需管理服务器等基础设施，只需编写代码并上传，程序员从底层设备维护中解放出来，只考虑实际业务逻辑即可。 |  维护成本高，自行维护服务器，需要处理服务器宕机、服务器扩容等一系列底层琐碎的事情   |
| 可用性  | 可用性高，函数计算为用户准备弹性、可靠的计算资源 |  服务器故障会对应用服务产生严重影响 |
| 费用  | 按需付费，只为实际使用的计算资源付费，代码未运行则不产生费用 |  需要支付服务器的费用，代码运行与否都要收费   |

### 架构设计
![基于阿里云函数计算的云相册](https://pitcures.oss-cn-hangzhou.aliyuncs.com/%E5%9F%BA%E4%BA%8EFC%E7%9A%84%E4%BA%91%E7%9B%B8%E5%86%8C.png)

### 主要部分
* **登录入口：**是存储在`OSS Bucket/login`目录下的一个静态登录页面，用户在这里输入用户名密码，确认登录后会通过API网关触发函数计算的鉴权函数`logAuth`
* **uploadLoginPage函数：**用于上传静态登录页面，只在初始化阶段运行一次
* **API网关：**用于触发后台鉴权函数`logAuth`
* **logAuth函数：**通过[API网关](https://www.aliyun.com/product/apigateway?spm=5176.8142029.388261.669.3836dbccT65LNV)获取用户在登录页面输入的用户名密码，并与[表格存储(OTS)](https://www.aliyun.com/product/ots?spm=5176.173847.765261.433.IhiaPO)中存储的用户名密码进行比对，实现鉴权功能
* **OSS Bucket photo-gallery/source：**[对象存储(OSS)](https://www.aliyun.com/product/oss?spm=5176.8142029.388261.452.2958d1efdAl1Aj)中的`photo-gallery`用于存储整个云相册的文件，其中`source`目录存储用户上传的图片原文件
* **resize函数：**由`OSS Bucket photo-gallery/source`触发，对用户上传的原图片进行`resize`，并将`resize`后的图片存储到`OSS Bucket photo-gallery`的`processed`目录下，即当`photo-gallery/source`有图片存入时，会触发`resize`函数
（事件触发是函数计算提供的一种机制）
* **OSS Bucket photo-gallery/processed：**用于存储裁剪后的图片
* **build函数：**由`OSS Bucket photo-gallery/processed`事件触发，将`OSS Bucket photo-gallery/processed`中的图片构造成html页面，生成美丽的相册页面，并将静态文件存储到`OSS Bucket photo-gallery/web`中
* **Time trigger：**由于oss图片及网页的访问权限是private的，那么用户认证成功后跳转到的链接是带有时间戳的oss的链接，当前时间大于时间戳的时间时，此页面会过期。为了防止用户看到过期的页面，我们采取的方式是1. 后台每2h定时触发`build`函数，生成新的相册页面存储到`OSS bucket`的`web`目录下，相册页面的有效期是8h；2. 同时前端会检测当前页面是否过期，页面距离过期还有半小时时弹出“页面已过期，请重新登录”的提醒，确定后弹出登录页面，重新登录，此时用户就可以看到函数定时触发新生成的页面。采用此种方法有效避免用户看到不友好的过期页面。
###  用户访问过程
* 用户在登录页面，输入用户名、密码，点击登录时，发送post请求到API网关，API网关触发`logAuth`鉴权函数，`logAuth`函数通过比对用户输入的用户名和密码与OTS中存储的用户名密码来对用户进行鉴权，如果是合法用户，即返回OSS中带有Signature的URL
* 合法用户通过`logAuth`函数返回的URL访问OSS中存储的静态页面，就可以看到美丽的相册啦；非法用户无法得到返回的URL，会继续停留在登录页面

### 图片存储过程
* 管理员将图片存储到`OSS Bucket photo-gallery`的`source`目录下，通过设置函数计算的触发器（[函数计算触发器参考文档](https://help.aliyun.com/document_detail/53097.html?spm=5176.doc51733.6.555.peJTrx)），source目录下一旦有数据上传会触发resize函数，对原图片进行裁剪，并将裁剪结果存入到`OSS Bucket photo-gallery`的`processed`中
* 同理，`OSS Bucket photo-gallery`的`processed`目录一旦有数据上传，会触发`build`函数，将`OSS Bucket photo-gallery/processed`中的图片构建成静态页面，并将结果存储到`OSS Bucket photo-gallery/web`中


## 搭建过程
### 准备阶段
**Attention：**：所使用的一起阿里云服务最好建在同一区域，避免带来不必要的问题
1. **开通服务**
分别去阿里云相应服务的主页开通以下几种服务[函数计算](https://www.aliyun.com/product/fc)、[OSS](https://www.aliyun.com/product/oss)、[SLS](https://www.aliyun.com/product/sls)、[OTS](https://www.aliyun.com/product/ots)、[API网关](https://apigateway.console.aliyun.com/)、[RAM控制台](https://ram.console.aliyun.com/#/user/list?guide)的服务（如果已开通相应服务，请跳过这步）
2. **新建子账户**
到[RAM控制台](https://ram.console.aliyun.com/#/user/list?guide)新建一个子账户，用户管理-新建用户，存下对应的`AccessKey`和`AccessSecret`。并对此用户授予相关权限，主要包括`AliyunOSSFullAccess`，`AliyunRAMFullAccess`，`AliyunOTSFullAccess`，`AliyunLogFullAccess`，`AliyunFCFullAccess`（RAM相关文档请参考[RAM官方文档](https://help.aliyun.com/document_detail/28627.html?spm=5176.doc28647.6.539.74rWLu)）
3. **新建OSS Bucket**
到[OSS控制台](https://oss.console.aliyun.com/overview)新建一个bucket，博主的bucket名字是`photo-gallery`，由于bucket name不可重复，所以你们要取其他未被占用的名字哈，然后新建四个目录，分别为`source`，`processed`，`web`，`login`，其中`processed`目录下有两个目录`fulls`和`thumbs`，这是出于相册的页面显示的需要，分别为大图和缩略图（OSS相关操作请参考[OSS官方文档]
4. **新建OTS Instance**
到[OTS控制台](https://ots.console.aliyun.com/index?spm=5176.54465.905680.btn5.24771229kwo6rs#/list/cn-shanghai)新建一个实例`instance`（个人理解OTS的实例就是MySQL中的database的概念），在这个实例里新建一个数据表`userTable`（数据表就是MySQL中的table）,数据表是用于存储访问控制的用户名username和密码password，主键是username。记录下`instanceName`和`tableName`，然后下载OTS的[客户端工具](https://help.aliyun.com/document_detail/44985.html?spm=5176.doc55220.2.7.J8foWD)，向数据表中写入合规的用户名和密码（OTS相关操作请参考[OTS官方文档](https://help.aliyun.com/document_detail/27280.html?spm=5176.54465.905680.btn4.24771229kwo6rs)）
![新建数据表](http://pitcures.oss-cn-hangzhou.aliyuncs.com/%E5%88%9B%E5%BB%BAtable.png)
![向数据表中插入数据](http://pitcures.oss-cn-hangzhou.aliyuncs.com/%E6%8F%92%E5%85%A5%E6%95%B0%E6%8D%AE%E5%88%B0table%E7%A4%BA%E4%BE%8B.png)
5.  **新建SLS Project**
去[SLS控制台](https://sls.console.aliyun.com/?spm=5176.55536.857803.control.7fccd522fjtnGz#/)新建一个`Log Project`和 `Logstore`，新建完成后，点击相应`logstore`的`查询`，右上角`开启索引`（SLS相关操作请参考[SLS官方文档](https://help.aliyun.com/document_detail/48869.html?spm=5176.55536.857811.1.7fccd522K3YjaC)）
![logstore查询](http://pitcures.oss-cn-hangzhou.aliyuncs.com/logstore%E6%9F%A5%E8%AF%A2.png)

![开启日志库索引](http://pitcures.oss-cn-hangzhou.aliyuncs.com/logstore%E5%BC%80%E5%90%AF%E7%B4%A2%E5%BC%95.png)

### 部署阶段
6. **新建FC Service**
到[函数计算控制台](https://fc.console.aliyun.com/overview)新建一个服务photo-gallery，开启高级角色，这里新建一个角色，并去[RAM控制台](https://ram.console.aliyun.com/#/role/list)角色管理->授权 为此角色授权，主要包括`AliyunOSSFullAccess`，`AliyunOTSReadOnlyAccess`
![新建Service](http://pitcures.oss-cn-hangzhou.aliyuncs.com/%E6%96%B0%E5%BB%BAservice.png)
![角色授权](http://pitcures.oss-cn-hangzhou.aliyuncs.com/%E8%A7%92%E8%89%B2%E6%8E%88%E6%9D%83.png)
7. **在本地配置[fcli](https://help.aliyun.com/document_detail/52995.html)**(已配置过fcli的用户请跳过此步骤)
	* 下载[fcli安装包](https://github.com/aliyun/fcli/releases)
	* 在fcli目录下，`./fcli shell`进入shell模式完成初始配置，详细信息参见[fcli](https://help.aliyun.com/document_detail/52995.html)。Tips：其中`access key id`和`access key secret`就是第2步创建的用户的AccessKey和AccessSecret
详情请参照[授权函数计算日志库写权限](https://help.aliyun.com/document_detail/52586.html?spm=5176.doc51733.6.554.Yz3piJ#%E6%8E%88%E6%9D%83%E5%87%BD%E6%95%B0%E8%AE%A1%E7%AE%97%E6%97%A5%E5%BF%97%E5%BA%93%E5%86%99%E6%9D%83%E9%99%90)
8. **通过fcli创建logAuth函数并创建API**
详情请参考[函数计算使用示例](https://help.aliyun.com/document_detail/51783.html?spm=5176.doc51733.6.552.lirLZD)
	* 首先需要下载并解压相应代码[photo-gallery.zip]() ，并将`config.json`文件中的配置改成自己的配置，按照示例和提示写就可以，其中`logAuthApiUrl`先不写，它代表的是通过API网关触发`logAuth`函数时的API网关地址。`config.json`中`domainName`以后是使用CDN缓存静态文件的配置信息，也先不填写。
	* 在fcli中进入你创建的Service，即`cd your-service`
	* 创建logAuth函数：
		* 创建函数`mkf logAuth -t nodejs6 -h login/logAuth.handler -d code`（code为code文件夹相对于fcli的可执行文件的位置）
	* 通过API网关触发logAuth鉴权函数
API网关触发函数计算详情请参考[以函数计算作为 API 网关后端服务](https://help.aliyun.com/document_detail/54788.html?spm=5176.product29462.6.585.1vpZ21)和[API网关+函数计算实践](https://yq.aliyun.com/articles/165104?spm=5176.173847.906745.9.II8bJO)
		*  创建API分组：在[API网关控制台](https://apigateway.console.aliyun.com/?spm=5176.56205.824509.a4.33c0e525iwLR4b#/cn-shanghai/groups/list)，分组管理->创建分组，新建API分组，会得到一个二级域名，记录下来
		*  创建API：API列表->创建API，后端服务信息选择函数计算，选择对应your-service的服务中的logAuth函数，然后发布线上
		*  API创建完毕以后，填写`config.json`中的`logAuthApiUrl`,为`API分组的二级域名/请求path`的形式，例如博主的是`http://922d3366e8d2421abe962408d613b800-cn-shanghai.alicloudapi.com/login`
![API分组截图1](http://pitcures.oss-cn-hangzhou.aliyuncs.com/%E5%88%9B%E5%BB%BAAPI%E5%88%86%E7%BB%84.png)
![API分组截图2](http://pitcures.oss-cn-hangzhou.aliyuncs.com/Blog%E5%9B%BE%E7%89%87/api%E5%88%86%E7%BB%84.png)
![API创建截图1](http://pitcures.oss-cn-hangzhou.aliyuncs.com/%E5%88%9B%E5%BB%BAAPI%20P1.png)
![API创建截图2](http://pitcures.oss-cn-hangzhou.aliyuncs.com/%E5%88%9B%E5%BB%BAAPI%20P2.png)
![API创建截图3](http://pitcures.oss-cn-hangzhou.aliyuncs.com/%E5%88%9B%E5%BB%BAAPI%20P3.png)
![API创建截图4](http://pitcures.oss-cn-hangzhou.aliyuncs.com/%E5%88%9B%E5%BB%BAAPI%20P4.png)

9. **创建其他函数**

* uploadLoginPageCDN函数：
	* fcli上执行：`mkf uploadLoginPage -t nodejs6 -h login/uploadLoginPage.uploadLoginPage -d code`	
* resize函数：
	* `mkf resize -t nodejs6 -h resize/resize.resize -d code -m 512`
	* 创建oss触发器：去[函数计算控制台](https://fc.console.aliyun.com/)，点击`resize`函数->触发器->创建触发器，如果提示需要授权，则点击授权
![resizeTrigger创建](http://pitcures.oss-cn-hangzhou.aliyuncs.com/resizeTrigger%E5%88%9B%E5%BB%BA.png)

* build函数（与resize函数类似）：
	* 创建函数：`mkf build -t nodejs6 -h site-builder/index.build -d code`
	* 创建oss触发器：
![buildTrigger](http://pitcures.oss-cn-hangzhou.aliyuncs.com/buildTrigger%E5%88%9B%E5%BB%BA.png)
10. **添加time trigger**
为build函数添加定时触发的time trigger。如图所示
![time trigger](https://pitcures.oss-cn-hangzhou.aliyuncs.com/time%20trigger.png)，其中触发消息即为oss的event（可以在代码执行->触发事件中编辑后粘贴过来）。（time trigger添加可参考[定时触发函数](https://help.aliyun.com/document_detail/68172.htmlspm=a2c4g.11186623.6.562.C7ntR6)）

### 测试阶段

#### 函数测试

1. **测试uploadLoginPage函数**
执行`uploadLoginPage`函数，看`OSS bucket`的`login`目录下是否有文件上传
2. **测试resize函数**
设置触发事件为oss模板，并将模板内的id和bucketName都改成自己的，执行函数
3. **测试build函数**
触发事件不变，执行函数
4. **测试logAuth函数**
获取`login`目录下的`index.html`页面的链接，这是整个相册的登录页面，输入正确or错误用户名密码，看是否会跳转到相册页面

#### 整体测试
1. 在`OSS bucket`下的`source`目录下新建一个目录Album1，在Album1中上传or删除图片，查看`processed/fulls`和`processed/thumbs`目录下是否有同名的文件上传or删除
2. 查看`web/homepageSite`目录下的`index.html`文件是否更新
3. 通过`loginUrl`登录页面，输入正确的用户名和密码，跳转到相册页面

### 其他问题
1. Q：由于`logAuth`函数返回给合法用户的是带Signature的OSS文件链接，这个链接是有一定时效的，链接过期了之后用户再从登陆页面登陆岂不是看到一个过期页面，这怎么处理？
    A：这里采用后台CronTab触发+检测过期时间并强制重新登录的方式。设置链接过期时间为8h,CronTab每隔2小时触发一次build函数，使得用户在得到此链接后至少6个小时是有效的。并且有一个js文件负责检测当前页面是否即将过期，如果过期时间在0.5h以内，会弹出弹框提醒重新登录，用户重新登录会得到一个新的有效的链接。所以，此种方式，用户不会看到不友好的过期页面
2. Q：向source目录下放文件可以触发整个函数运行，那删除文件可以吗？
    A：删除文件是否会触发函数运行取决于用户配置的触发器与函数代码本身，一方面触发器中要添加删除操作的触发事件，另一方面代码中要检测触发事件类型，并进行相应处理，本示例支持当`source`目录下删除文件时，`prosessed`目录下同时删除同名文件，并重新建立静态页面

## 参考文献
[Building a serverless password-protected photo gallery](http://www.jpsim.com/awspics/)
[Animated login form](https://codepen.io/boudra/pen/YXzLBN?editors=1111)
[HTML5 site templates](https://html5up.net/)
[函数计算官方文档](https://help.aliyun.com/document_detail/52895.html?spm=5176.doc53097.6.539.O3rJZ5)
[对象存储官方文档](https://help.aliyun.com/product/31815.html?spm=5176.750001.2.11.jcR6sV)
[API网关官方文档](https://help.aliyun.com/document_detail/29464.html?spm=5176.doc54788.6.539.0RJZwZ)
[表格存储官方文档](https://help.aliyun.com/document_detail/27280.html?spm=5176.54465.905680.btn4.24771229kwo6rs)
[访问控制官方文档](https://help.aliyun.com/document_detail/28627.html?spm=5176.doc28647.6.539.74rWLu)

**That's all，enjoy it~**
Any question，可留言，或加入函数计算官方客户群（钉钉群号：11721331）