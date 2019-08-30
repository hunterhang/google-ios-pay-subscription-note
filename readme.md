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

### 二、订阅购买（subscription）
#### 1. 特性票据
####【Apple】
1. Apple 有两种票据，APP 都可以拿得到，一种短票据（约4KB，不会增长，ios6之前的票据），一种长票据（ios6 之后，会无限增长，包含了用户的所有购买记录，几百KB都有可能）；长票据可以根据Appstore的中文文档的描述解开，拿到详细内容，但是过于复杂，一般不这么干。由于长票据太大，云端不容易存储，所以云端只会存储短票据；
2. Apple 的每一次续费，都会产生一个票据，任何一次票据都可以查询到当前的最新状态；如果当前是最新票据，再/VerifyReceipt 接口会返回receipt_info和receipt 两个字段，一个是json结构，一个base64结构，其实质一样；如果当前票据不是最新票据，则该接口会返回 latest_receipt 和 latest_receipt_info 两个结果；如果当前票据已经过期，则该接口会返回 latest_expires_receipt 和 latest_expires_receipt_info 结构体；
3. Apple 的票据包括两个关键字段original_transaction_id 和 transaction_id ，分别表示原始订阅ID和当前的子订阅ID；第一次购买时，两个字段值一样的；
一个Appstore 帐号对某件唯一商品，只有存在一个original_transaction_id；
4. Apple 首次购买时，用户在AppStore 中无法订阅，只能在应用内购买；一但在应用内订阅以后，用户即可以在appstore 中看到已过期和未过期的订阅，也可以进行续费，如果用户在appstore 中续费，则没有办法调用云端的接口来创建订单号，此时只能通过appstore的通知机制来完成，下文中会讲到。当然，用户在下次打开APP时，会在keychain 中拿到最新的购买票据。
5. Apple的事件通知中，首次购买，过期，取消，回复购买都有明确的通知，但是续费事件，没有通知，此时，如果业务想要自己平台订单，则无法处理。因此，在用户首次购买时，需要做几件事件：
- 存储pay_token; 
- 根据pay_token获取订阅信息，获取到expires_time的字段，插入数据库中一个异步任务，在下次（expires_time - 24小时）时，查询该用户的当前状态，如果transaction_id 发生了变化，则表示用户续订了；否则等过期事件（ios 的过期事件时，发过来的是用户状态变化的事件，此时会多一个latest_expires_receipt 字段的通知）。为什么提前24小时查询用户状态？因为ios 会提前24小时开始尝试扣费，直到expries_time 过期 ，则放弃扣费。
- Apple 可以没有试用期，而Goole 必须有试用期，试用期是免费的。
- APP 未通过审核前，无法在appstore中看到需要购买的商品，也无法暂停订阅。因此，为了方便测试，app内的商品，都是云端后台自己控制的，与appstore中配置的保持一致即可；这样，可以让测试人员看到新的商品，而非灰度用户则无法看到。 新商品上线，也是需要官方审核的。
6. appstore 做订阅事件通知时，只需要配置一个回调URL即可，没有其它特殊操作；
7 调用/VerifyReceipt 接口时，需要带上password 参数。

####【Google】
1. Google play的票据不会发生变化的；根据票据可以获取得到order_id，可以分离出父order_id和子order_id ；当用户的订阅已经过期以后，则重新购买，此时父order_id 也会发生变化 ；
2. Google play 只能在应用内购买商品，在goole play可以暂停订阅等操作，但无法购买商品；在测试时，也可以在google play中看到对应的操作，而appstore则不行；
3. Google play的事件比较全，其本不需要特殊处理。
4. Google play在有很多接口，都需要认证，调用时，需要带上access_token。 而获取access_token是通过refresh_token来刷新的，这个通过google 的相关配置可以获取得一以refresh_token和client_id，然后做线程定时刷新即可，refresh_token没有过期时间；
5. Google play在配置回调地址时，比较麻烦，还需要验证服务的网址合法性。也可以选择jwt_token认证，也是比较复杂，也可以不使用jwt_token认证。
6. google 还有几个其它的API接口，例如：即款，取消，确认等，业务可根据需要实现。

#### 业务事件的抽象
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


