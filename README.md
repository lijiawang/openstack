### `openstack`新建云主机流程介绍
`openstack` 可以登录到你的`dashboard`上之后点击“项目-计算-实例-创建实例”按钮，然后选择一下相应的配置，只短短十几秒内就可以创建好一台云主机供我们使用了，然而这么牛逼的事情是怎么做到的呢？
新建一个云主机流程图
![](https://ljw.howieli.cn/blog/2017-7-3/%E5%88%9B%E5%BB%BA%E8%99%9A%E6%8B%9F%E6%9C%BA%E6%B5%81%E7%A8%8B%E5%9B%BE.png)

上图我们可以看出要想新建一台完整的云主机至少需要29步。
### 流程图分析
#### 流程-1
首先我们访问`dashboard`之后，浏览器会显示一个登陆界面（如下图所示），`horizon`会让我们登陆用户名和密码，然后`horizon`会把用户名和密码交给`keystone`来确认我们的身份，`keystone`确认成功后，才会允许我们登陆进去。（在`openstack`里`keystone`就相当于`openstack`这些组件共同的老大哥）
![](https://ljw.howieli.cn/blog/2017-7-3/%E7%99%BB%E9%99%86%E7%95%8C%E9%9D%A2.png)
在原来我一直以为这里登陆的时候只不过是一个简单的`django`框架使用`pymysql`直接查询数据库，而实际上这里的表单信息是提交到了`keystone`，然后通过`keystone`查询数据库进行验证的

#### 流程-2
`keystone`接受到`horizon`前端表单传过来的用户、密码信息以后，会去查询数据库来确认身份，确认了身份后会将一个`token`（就相当于办理了一个身份证~~~~）返回到该用户，让这个用户在进行操作的时候不在需要提供用户名和密码了，而是直接拿出`token`来。
#### 流程-3
`horizon`拿到`token`之后，实际上这里在`web`页面上的显示就是登陆成功了，接下来就是找到“项目-计算-实例-创建实例”按钮，点击创建实例，然后根据我们的需求填写云主机相关的配置信息。
填写完配置信息后，点击启动实例之后，`horizon`就会带着三个东西去找`nova-api`了。
- 创建云主机的请求
- 云主机相关配置信息
- 刚刚`keystone`返回给的`token`

#### 流程-4
`nova-api`会拿着`horizon`给的`token`去找老大哥`keystone`验证去。
#### 流程-5
`keystone`看到`nova-api`拿来的`token`，查询一下数据库确认，然后告诉给`nova-api`，这兄弟信得过，可以照他的安排去做。
#### 流程-6
`nova-api`从大哥那回来，接收了`horizon`提供的两样东西，一是云主机配置信息，二是创建请求，这`nova-api`手底下也有一帮小兄弟，这帮人之间沟通可不太方便都得通过一块小黑板（`mq`消息队列），把自己的需求写在小黑板上，能做的了这事的人自然就去做了。但配置信息现在还不能写在小黑板上，得找到确定去干活的人之后才行啊，所以`nova-api`就把配置信息放到数据库里。
#### 流程-7
数据库把配置信息收好之后，对`nova-api`说了声，我放好了。
#### 流程-8
放好配置信息后`nova-api`就在小黑板上写“现在要创建一台云主机，配置信息我已经放到数据库了，然后让`nova-schedular`给安排。
#### 流程-9
`nova-schedular`他就像是`nova-api`的秘书，`nova-api`的有事都是通过它交代给其他人的，这一步就是他从小黑板上看到了`nova-api`的信息。
#### 流程-10
`nova-schedular`现在知道了要创建云主机，但它要看一看云主机都要什么配置，才好决定该把这事交给谁去做（这里是指多个`nova-compute`的情况，各个计算节点的资源使用情况都在`nova-schedular`这里），所以他让数据库把云主机配置信息发给他看看。
#### 流程-11
数据库收到请求之后，把云主机配置信息发给`nova-schedular`。
#### 流程-12
`nova-schedular`拿到配置信息后，使用调度算法决定了要让`nova-compute`去干这个事，就在小黑板上写“`nova-compute`你给创建个云主机，配置都在数据库里了”。
#### 流程-13
`nova-compute`看到小黑板上的东西之后，本应该直接去数据库拿取配置信息，但因为`nova-compute`的特殊身份，`nova-compute`所在计算节点上全是云主机，万一有一台云主机被黑客入侵从而控制计算节点，直接拖库是很危险的。所以不能让`nova-compute`知道数据库在什么地方。
#### 流程-14
`nova-compute`没办法去数据库取东西难道就不工作了吗？那可不行啊，他不知道去哪取，但他哥们知道啊，于是他在小黑板上写“`nova-conductor`，你帮我去数据库取一下配置信息”。
#### 流程-15
`nova-conductor`从小黑板上看到了`nova-compute`的请求。
#### 流程-16
`nova-conductor`告诉数据库我要查看某某云主机的配置信息。
#### 流程-17
数据库把云主机配置信息发送给`nova-conductor`。
#### 流程-18
`nova-conductor`把配置信息写在小黑板上。
#### 流程-19
`nova-compute`从小黑板上读取云主机的配置信息。
#### 流程-20
`nova-compute`拿到了云主机配置信息一看，人家可是专业的，立马就知道该怎么做了，先去找`glance-api`拿镜像吧，刚才讲了那么多，可都是在`nova`组件内部的，这次去找别的组件可不是写在小黑板上了，它得带着自己的身份证去，告诉`glance-api`，我要`xxx`镜像。
#### 流程-21
`glance-api`看`nova-compute`过来，他可不认识`nova-compute`，让`nova-compute`拿出身份证，拿着人家身份证找到自己的老大哥`keystone`看看这人靠不靠谱，`keystone`一看，没问题，按他说的做吧（在`nova`验证`horizon`被当做两步，这里化做一步，是为了简化重复的流程）。
#### 流程-22
`glance-api`把镜像资源信息返回给`nova-compute`（这里主要说创建云主机的过程，除`nova`外其他组件内部先不提）。
#### 流程-23
接着`nova-compute`找到`neutron-server`（图里画错了，因为是偷别人的图，自己的图画了半天画不好。。。），告诉他我要`xxx`网络资源。
#### 流程-24
`neutron-server`也不认识他，拿着他的身份证找`keystone`确认了一下身份。
#### 流程-25
`nuetron-server`把网络资源信息返回给`nova-compute`。
#### 流程-26
`nova-compute`找到`cinder-api`要存储资源，云主机得有硬盘啊，得存东西啊（同样，这里图中也有错误）。
#### 流程-27
`cinder-api`也不认识他，拿着他的身份证找`keystone`确认了一下身份。
#### 流程-28
`cinder-api`把存储资源信息返回给`nova-compute`。
#### 流程-29
`nova-compute`拿到了所有资源之后，他其实也只是个收集信息的，他把工作全都交给了真正创建虚拟机的`Hypervisor`（`kvm`，`xen`等虚拟化技术）。

到此为止，你已经拥有了一台云主机了，流程看似复杂实际上在几十秒就完成了。
























































