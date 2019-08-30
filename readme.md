# 本文描述了google play 和 appstore的订阅支付相关问题
### 一、一次性购买
一次性购买的流程比较简单，流程基本相似。
##### Appstore
1. 用户点击“购买”；
2. APP 调用云端接口，创建订单，获取订单ID(order_id)；
3. APP 调用AppStore的API 完成支付，并获取得到receipt(Apple) 和 purchase_token(GoolePlay)，以前将支付token统称“pay_token”.
4. APP 将pay_token,order_id，user_id(用户购买前先登录) 发送到云端验证 ；
5. 云端将pay_token发给平台验证，平台验证pay_token有效后，再根据pay_token来更新order_id的状态；
6. 云端通知业务方发货，完成一性次购买流程。
* 注意：在第4步中，用户可以拿到A用户的pay_token，和B用户的order_id 来验证。此时，云端会验证pay_token有效，给B用户发货币了。由于google 有透传字段的机制，可以避免该问题，而apple则没有，需要业务自己做去重，标识该pay_token已经被使用过。

##### Appstore 与 Google Play的不同点
1. Appstore 的票据非据很长，有4KB左右；Google play的pay_token 只有 256 字节；
2. Google play 购买物品时，设置一个order_id，购买完成后，调用云端接口验证票据时，可根据pay_token获取购买信息时（/get） 接口时，返回order_id（不一定时order_id，可以设置一个大的字符串）；而AppStore 则不行。 如果有透传字段，则APP完成购买以后，只需要将pay_token发到云端，云端即可知道，对应的order_id。

##### 其它 
1. 一次性购买，可以购买多次，后台自动累加。

### 一、事件
##### 【开通】事件
1. Apple 
- 首次订阅，客户端会调用verify接口，后台判断是否为首次购买（original_transaction_id == transaction_id），如果是，则插入一条事件到DB，表示开通事件。
- 过期后，再次购买（有可能在appstore续费的，不一定走verify接口），后台在收到notice(用户状态变化)后，查询用户的当前状态，如果为过期？而此次的receipt对应的过期时间是在未来？则表示用户重新开通了。此时，插入一条事件到DB，表示开通。

2. Google
- 首次开通，根据事件走。

##### 【续费】事件
1. Apple
- 异步扫描的进程，每5秒扫描一次DB，将最近要处理的事件，拿出来查询其状态，并处理掉。 在用户购买以后，立即可扫描出【开通】的事件，回调订阅服务，并发出开通事件。在调用成功以后，删除该【开通事件】，插新插入一条【续费】事件，并设置下次检查时间为 expires_time - 24 小时。

2. Google
- 根据事件判断【续费】事件

##### 【过期】事件
1. Apple
- 异步扫描的进程，在扫描到【续费】事件时，每小时扫描一次，当扫描到expires_time 过期了，则判断为【过期】事件。

2. Google 
- 根据事件判断【过期】事件

##### 【取消】事件
1. Apple 
- 当收到用户状态变化的Apple Notice事件时，判断用户之前的状态为 【续订】，而本次receipt对应的状态为【未续订】，则可判断本次为取消续订事件。
2. Google
- 根据事件判断【取消订阅】

##### 【退款】事件
1. Apple
- 根据事件来判断
2. Google 
- 根据事件来判断 

##### 【继续续订】事件
1. Apple
- 当收到用户状态变化的Apple Notice事件时，判断用户之前的状态为【取消】，而本次receipt对应的状态为【续订】，则可判断本次为继续续订事件。
2. Google
- 根据事件来判断


