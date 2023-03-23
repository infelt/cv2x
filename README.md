# 腾讯车路协同V2XLib接口文档


# 一、简介
​		V2XLib SDK集成了完整的V2X服务功能，包含周车预警、换道识别、消息推送、红绿灯预警等功能模块，支持http/https、ws/wss协议，确保长连接稳定。并提供简单的API方便开发者使用。利用SDK调用能保证更好的实时性和稳定性。

# 二、集成方式
## 2.1 添加maven地址

​	在工厂根目录build.gradle或setting.gralde中，添加maven地址（外网请添加外面地址）。

```groovy
allprojects {
    repositories {
        //...
        //内网
        maven {
            url 'https://mirrors.tencent.com/repository/maven/cv2x-snapshots'
            //url 'https://mirrors.tencent.com/repository/maven/cv2x'
        }
        //外网
          maven {
            url 'https://raw.githubusercontent.com/infelt/cv2x/main'
        }
    }
}
```

## 2.2 添加依赖

在对应模块build.gradle中添加依赖。

```groovy
//v2x sdk
implementation "com.tencent.v2xlib:v2xlib-simplify-release:1.0.1.24-SNAPSHOT"
//v2x模拟测试回放sdk
implementation "com.tencent.grpc:replay-release:1.0.1.24-SNAPSHOT"
```





# 三、api 介绍

## 3.1 调用步骤
  - 权限申请(定位权限)
  - 配置V2XConfig参数
  - 初始化V2XModule,调用V2XModule.init()函数
  - 注册登录（公网模式需要此步骤，本地obu模式可省略）(ILoginManager)
  - 启动v2xServer,调用V2XModule.start()函数
  - 注册监听回调
  - 释放资源

## 3.2调用示例

### 3.2.1 权限申请
```java
//也可自行申请，保证app有定位权限即可

if (PermissionUtil.requestMustPermissions(this, PermissionUtil.PERMISSION_REQUEST)) {
    initV2xServer();
}

//在权限回调中也需要调用初始化
@Override	
public void onRequestPermissionsResult(int requestCode, @NonNull String[] permissions,
                                        @NonNull int[] grantResults) {
    super.onRequestPermissionsResult(requestCode, permissions, grantResults);
    Log.d(TAG, "onRequestPermissionsResult " + requestCode);
    if (requestCode == PermissionUtil.PERMISSION_REQUEST && PermissionUtil.checkMustPermission(this)) {
        initV2xServer();
    } else {
        Toast.makeText(this, "app缺少必要的权限", Toast.LENGTH_LONG).show();
        finish();
    }
}

```

### 3.2.2 初始化配置，及初始化v2x服务
```java
V2XConfig.setIsLog(false);
//...
V2XConfig.setControlServerURL("http://xxx:4245/api/getServer");

V2XModule v2xModule = V2XModule.getInstance();
v2xModule.init(this);
  
```

### 3.2.3 注册登录(ILoginManager)

#### 1、手机号登录

- 判断是否已经登录（isUserLogin）
- 获取验证码（getCode）
- 校验验证码（checkCode）
- 根据校验返回信息中是否存在用户信息，判断是否是已注册用户
- 已经注册，登录成功
- 未注册，调用注册接口注册（registerUserInfo）

#### 2、微信登录

- 登录微信官网申请appkey和appsecret

- 在对应的moudle中添加微信登录依赖，`api "com.tencent.mm.opensdk:wechat-sdk-android:+"`

- 在包名相应的目录新建wxapi 目录，并新建WXEntryActivity继承于BaseWXEntryActivity。并在manifest中注册。并添加queries标签。

  ```xml
  
  <queries>
  	<package android:name="com.tencent.mm" />   // 指定微信包名
  </queries>
  
  <activity
      android:name=".wxapi.WXEntryActivity"
      android:label="@string/app_name"
      android:theme="@android:style/Theme.Translucent.NoTitleBar"
      android:exported="true"
      android:taskAffinity="填写你的包名"
      android:launchMode="singleTask">
  </activity>
  
  <!-- 参考文档https://developers.weixin.qq.com/doc/oplatform/Mobile_App/Access_Guide/Android.html-->
  ```

- 在登录的modle中初始化微信登录回调接口。

  `WxLoginModel.getInstance().setWxLoginCallBack(this)`

- 开启微信授权登录,传入appid，和secret。

  ```java
  WxLoginModel.getInstance().startWxAuthReq(TxLoginActivity.this,
          AppConfig.APPID_WEIXIN, AppConfig.APP_SECRET_WEIXIN);
  ```

- 监听setWxLoginCallBack设置的回调

  ```java
  @Override
  public void onWxLoginSuccess(LoginUserData loginUserData) {
      //登录成功，保存用户信息，页面跳转
  }
  
  @Override
  public void onWxLoginFailed(int i, String s) {
    //登录失败
  }
  
  @Override
  public void onNeedRegister(UserInfo userInfo) {
     //用户未注册，跳转注册页面进行注册。
  }
  ```

  #### 

#### 3、匿名登录

- 获取能标识用户的唯一ID

- 注册（registerUserInfo）

  返回成功则说明成功注册，返回失败且errorCode =  V2xErrorConstants.ErrorUserExists，则说明用户已经注册。

```java
     //注册信息中TypeID取值V2xUserConstants.TYPEID_ANONYMOUS
     userInfo.TypeID = V2xUserConstants.TYPEID_ANONYMOUS
     //OpenID赋值唯一id
     userInfo.OpenID = 唯一标识Id
```

- 根据返回如果已经注册则调用登录（checkUserInfo）

  参数openId，唯一标识ID。

#### 4、错误码说明（V2xErrorConstants）

```java
    //token非法
    int ErrorTokenInvalid = 4013;
    //用户已存在（已注册）
    int ErrorUserExists = 4014;
    //缺少OpenId
    int ErrorOpenIDIsRequired = 4015;
    //缺少手机号
    int ErrorPhoneIsRequired = 4016;
    //未知的typeId
    int ErrorTypeIDUnknown = 4017;
    //身份证号已存在
    int ErrorIDNumberExists = 4018;
    //数据库操作失败
    int ErrorDBOperateFailure = 4019;
    //微信登录时获取openid失败
    int ErrorRequestOpenIDFailure = 4020;
    //用户不存在
    int ErrorUserNotExists = 4021;

    //微信未安装
    int ERROR_WX_NO_INSTALL = -1001;
```





### 3.2.4 启动v2xServer

```java
V2XModule v2xModule = V2XModule.getInstance();
v2xModule.start();
//注册碰撞预警信息回调函数
v2xModule.setV2XCollisiionInfoListener(collisionData -> {
    //...
});
//注册交通信号灯信息回调函数
v2xModule.setV2XTrafficLightInfoListener(false, new TrafficLightInfoListener() {
    @Override
    public void onTrafficLightData(TrafficLightData trafficLightData, String s) {
        //v1版本服务器的交通信号灯消息回调
    }

    @Override
    public void onTrafficLightDataV2(TrafficLightDataV2 trafficLightDataV2, String s) {
        //v2版本服务器的交通信号灯消息回调
    }
});
//注册消息面板回调函数:图片、文字
v2xModule.setV2XWebInfoListener(infoListener -> {
    //...
});
//注册websocket连接成功监听
v2xModule.setV2XWebSocketOpenListener(webSocket -> {
    //...
});
//注册周车预警服务
v2xModule.setV2XAroundInfoListener(false, (aroundData, s) -> {
  //周车预警消息服务
});
```

### 3.2.5 结束释放资源
```java
//结束释放V2X服务
public void releaseV2Xserver(){
    V2XModule v2xModule = V2XModule.getInstance();
    v2xModule.stop();
}
```

## 3.3 api简介

### 3.3.1 **PermissionUtil，权限相关类**

1. ``boolean requestMustPermissions(Activity activity, int requestCode)``  

    用于申请sdk运行所必须的权限，如果已经有全部所需权限则返回ture；否则返回false,并将申请权限。

2. ``static final int PERMISSION_REQUEST = 1001``

    常量用于申请权限时的请求码。

### 3.3.2 **V2XConfig，配置相关类**

1. `` setRequestTimer(long updateCarInfoTimer) `` 
  
    用以设置内部获取gps的频率及发送主车位置信息频率，单位ms，默认值1000ms
   
2. `` setUpdateServerTimer(long getServerTimer)``

    用以设置刷新V2X边缘服务器地址的时间间隔，单位ms，默认值5000ms

3. ``setLaneLocationEnable(boolean laneLocationEnable)``

    设置是否开启车道定位功能。利用陀螺仪等传感器识别换道行为，默认开启

5. ``void setIsLog(boolean isLog)`` 

    设置是否记录日志信息。日志信息较多，建议仅在debug阶段开启，默认开启

6. ``void setCarType (int carType)`` 

    用以手动设置车辆类型,
    车辆类型
    1:一般车辆 4:卡车 5:警车 8:工程车 13:摩托车 14:巴士

7. ``void setControlServerURL(String controlURL)``

    用以设置云控中心服务器URL地址,云控中心服务器URL地址，默认端口4245。


8. ``void setTestServerURL(String testURL)``

     用以设置测试服务器URL地址,默认端口4246。该API仅在开发阶段开发使用

9.  ``void setLocalCompute(boolean localCompute)``
    
    用以设置是否启用本地计算能力, 参数：true：启用本地计算 ，false : 启用云端计算能力。
    默认为false启用云端能力。

10. `` void setUseCustomGps(boolean useCustomGps)``

    设置是否使用外部Gps数据。参数：true：启用外部gps数据 ，false : 启用本地数据。 默认为false。
    
    若设置为true,则需调用``offerCustomGpsInfo(GPSInfo gpsInfo)``来提供gps数据。

11. ``void setPostTypeWgs84(boolean usingWgs84)`` 

    设置是否采用GPS信息返回周车主车等位置信息。（目前适用于“周车预警”回调中），返回gps数据格式和上报车数据格式有关。

    参数 usingWgs84: true ,采用GPS数据 ; fasle,采用xyz相对位置格式。

12. ``void setCollisionType(int type)``

    设置预警消息类型，默认是普通预警。

    参数type：
    
    V2xConstants.WS_TYPE_NORMAL_COLLISION 普通预警；
    V2xConstants.WS_TYPE_AROUND_COLLISION 周车预警；
    
13. ``void setSendGsp02(boolean use02)``

      设置当前发送的gps坐标系，仅作用SDK内部获取的gps。外部传入的需自行转换。

      参数use02： false 默认发送84坐标系， ture 会将84转为02坐标发送。
    
14. ```
      boolean setLoggerWriterStrategy(ILoggerWriter writerStrategy)
    ```

​		设置日志保存策略，设置后v2x日志将不在保存日志到本地。		

15. ```
    void setForegroundService(boolean isForegroundService)
    ```

​		设置是否启动时服务是添加通知栏保持为前台服务，默认v2x启动后会显示前台通知。



### 3.3.3 **V2XModule ，模块初始化，设置接口回调**

1. ``V2XModule getInstance(Context context)``
   
    获取V2XModule实例以及初始化V2XModule

2. ``boolean start() ``
   
    启动V2X服务,在V2XConfig配置完成后调用

3. ``void stop()``
   
    结束V2X服务，并释放资源。在结束使用v2x或app退出时调用。

4. ``boolean offerCustomGpsInfo(GPSInfo gpsInfo)``

    用户自主注入gsp信息。需先设置`V2XConfig.setUseCustomGps(true)`。

    **GPSInfo 结构信息：** 

    | 参数名 | 说明 | 备注
    | :--- | :---: | :---: |
    | lon | 精度 | 必要 |
    | lat | 维度 |  必要 |
    | altitude | 海拔 |  非必要 |
    | speed | 速度 |  必要 |
    | heading | 方向角 |  必要 |
    | ptcType | 车辆类型 |  必要 |
    | timestamp | 时间戳 | 必要 |

5. ``void setV2XCollisionInfoListener(boolean trnTrm, CollisionInfoListener infoListener)``

​	设置碰撞预警信息回调函数。

​	参数 trnTrm: true 采用消息透传方式回调数据，false 采用反序列化后的对象回调数据。 


6. ``setV2XTrafficLightInfoListener(boolean trnTrm, TrafficLightInfoListener infoListener)``

    设置交通灯信息回调函数。

    参数 trnTrm: true 采用消息透传方式回调数据，false 采用反序列化后的对象回调数据。 其中TrafficLightInfoListener有两个回调方法，分别对应v1和v2版本。在设置服务器地址时会自动区分服务器版本。

7. ``void setV2XWebInfoListener(WebInfoListener infoListener)`` 

    注册消息面板信息回调函数

8. ``void setV2XVideoInfoListener(VideoInfoListener infoListener)``

    注册获取视频直播流URL的回调函数

9. `` setV2XAroundInfoListerner(boolean trnTrm, AroundInfoListener infoListener)``

    注册获取周车预警消息

    参数 trnTrm: true 采用消息透传方式回调数据，false 采用反序列化后的对象回调数据。

10. `` void setPointEventListener(IPointEventListener pointEventListener)``

    注册点事件消息回调（目前默认返回前向1.5km范围内点事件）
    
11. ```
     void setV2xErrorListener(IV2xErrorListener v2xErrorListener)
    ```

​		设置v2x内部异常监听，目前仅支持当后台服务用户清空后用户登录失效回调。



###  3.3.4 **回调接口说明**

#### 1、CollisionInfoListener

预警信息回调接口,需要实现`void onCollisionData(CollisionData colData, String msg)`方法。

其中若在设置回调函数``setV2XCollisionInfoListener ``中参数``trnTrm``设置为ture,则只返回CollisionData序列化后的数据msg,否则只返回对象colData。

| 参数名 | 说明 | 
| :--- | :---: |
| orientation | 预警方向 1:正前方向; 2:正后方向; 3:正右方向; 4:正左方向; 5:右前方向; 6:右后方向: 7:左前方向; 8:左后方向 | 
| coltype |碰撞类型 0: 正常 1: 有碰撞风险 2: 紧急制动 3: 异常车辆 4.:车辆失控 5: 超载 6: 逆向 7: 变道 8: 他车汇入 9: 他车汇出 10: 本车汇入 11: 本车汇出 12: 向右换道 14: 盲区预警 15: 紧急制动 16: 即将闯红灯 17: 已闯红灯 18: 左转辅助 19: 异常慢行 20: 绿波 21:异常车辆（非 who am i） 22.紧急制动（非 who am i） 23: 弱势交通参与者（非 who am i） |
| objtype |目标类型 0: 不明物体 1: 小汽车 2: 行人 3: 动物 4.:卡车 5: 警车 6: 落石 7: 救护车 8: 道路运维车辆 9: 事故目标 10: 道路施工 11: 抛撒物 12: 自行车 13: 摩托车 14: 公共汽车 15. 交通拥堵 16. 小雨 17. 大雨 18. 团雾 19. 消防车 20. 深坑 21. 积水 22. 限速 23. 限高 24. 限宽 25. 道路湿滑 26．急转弯 27. 无人机 28. 无人船 29. 恶劣天气 30. 交通管制 31. 解除限速 32. 信号灯 33. 限重提醒 34. 禁止左转弯提醒 35. 禁止右转弯提醒 36. 禁止直行提醒 37. 禁止掉头提醒 38. 禁止超车提醒 39. 禁止停车提醒 40. 禁止鸣喇叭提醒 41. 前方学校提醒 42. 事故易发路段提醒 43. 前方道路变窄提醒 44. 结冰预警 |
| distance |距碰撞物距离（限高/宽/速），当为绿波时，表示绿波速度(km/h) |
| level |预警级别，预警级别 0:无预警; 1:黄色预警; 2:橙色预警; 3:红色预警; |
| flag |遮挡类型 0: 无遮挡车辆（默认） 1: 被大车遮挡 2: 被小车遮挡 |
| extra |附加信息 |
| sendtime |消息发送时间戳(ms)，1970纪元后经过的毫秒数 |

ps: 具体预警组合类型参见附表1、附表2。

#### 2、TrafficLightInfoListener

交通灯信息回调接口，需要实现`void void onTrafficlightData(TrafficlightData trafficLightDataV2, String msg)`方法

其中若在设置回调函数``setV2XTrafficLightInfoListener ``中参数``trnTrm``设置为ture,则只返回TrafficlightData序列化后的数据msg,否则只返回对象trafficlightData。

| 参数名 | 说明 | 
| :--- | :---: |
| lat | 红绿灯纬度(WGS84坐标) |
| lng | 红绿灯经度(WGS84坐标) |
| ele | 红绿灯海拔，单位米 |
| heading | 红绿灯朝向的方向角；范围为0-360度，正北为0度，顺时针旋转 |
| sid | 红绿灯设备唯一id |
| time | 消息发送时间戳(ms)，1970纪元后经过的毫秒数 |
| laneID | 当前车道可能的行驶方向 1: 掉头; 2: 左转; 3: 直行; 4: 右转 |
| back/left/right/straight.status | 逆转/左转/右转/直行灯状态 -1:无此设备 0: 设备正常 1: 设备故障 |
| back/left/right/straight.color | 逆转/左转/右转/直行灯颜色 1: 绿灯 2: 红灯 3: 黄灯 |
| back/left/right/straight.time | 逆转/左转/右转/直行灯剩余时间，单位为秒 |
| distance | 离红绿灯或者路口的距离 |
| advice | 各个方向通行意见 |
| lane | 通行方向 1: 掉头; 2: 左转; 3: 直行; 4: 右转 |
| message | 通行意见 |

#### 3、WebInfoListener

消息面板回调接口，需要实现`void onWebInfoData(WebMsgData webMsgData)`方法

| 参数名 | 说明 | 
| :--- | :---: |
| txtMsg.txtNum | 文本消息个数 |
| txtMsg.texts | 文本消息列表 |
| imgMsg.resType | 资源类型 1:网络资源 2:本地资源 |
| imgMsg.imgUrl | 图片地址 |
| voiMsg.resType | 资源类型 1:网络资源 2:本地资源 |
| voiMsg.voiUrl | 语音资源地址 |
| voiMsg.support | 是否支持语音 |

#### 4、VideoInfoListener 

视频直播流URL的回调接口，需要实现`void onVideoData(VideoWebMsg videoWebMsg)`方法。
`videoWebMsg` 视频信息，其内部包含一个VideoInfo的集合。

| 参数名 |           说明           |
| :----: | :----------------------: |
|  dist  | 当前位置与摄像头位置距离 |
|  lat   |      摄像头维度信息      |
|  lng   |      摄像头经度信息      |



#### 5、AroundInfoListener 

周车预警消息回调接口，需要实现`void onAroundData(AroundData aroundData, String msg)`方法。

其中若在设置回调函数``setV2XAroundInfoListener ``中参数``trnTrm``设置为ture,则只返回结构体类型数据msg,否则只返回对象aroundData。 

##### 5.1 AroundData数据结构

| 参数名 | 说明 |
| :--- | :---: |
| mainCar | 主车信息 |
| otherobjs | 周车信息 |
| x |相对坐标x |
| y | 相对坐标y |
| z | 相对坐标z |
| head | 车辆行驶方向角（弧度-π - +π） |
| speed | 车辆行驶速度（m/s） |
| objtype | 目标类型 0: 不明物体 1: 小汽车 2: 行人 3: 动物 4.:卡车 5: 警车 6: 落石 7: 救护车 8: 道路运维车辆 9: 事故目标 10: 道路施工 11: 抛撒物 12: 自行车 13: 摩托车 14: 公共汽车 15. 交通拥堵 16. 小雨 17. 大雨 18. 团雾 19. 消防车 20. 深坑 21. 积水 22. 限速 23. 限高 24. 限宽 25. 道路湿滑 26．急转弯 27. 无人机 28. 无人船 29. 恶劣天气 30. 交通管制 31. 解除限速 32. 信号灯 33. 限重提醒 34. 禁止左转弯提醒 35. 禁止右转弯提醒 36. 禁止直行提醒 37. 禁止掉头提醒 38. 禁止超车提醒 39. 禁止停车提醒 40. 禁止鸣喇叭提醒 41. 前方学校提醒 42. 事故易发路段提醒 43. 前方道路变窄提醒 44. 结冰预警 |
| sid | 车辆sid |
| roadId | 道路id |
| secId | 路段id |
| laneId | 车道id |
| collision | 预警信息 |
| level | 预警级别 0:无预警; 1:黄色预警; 2:橙色预警; 3:红色预警; |
| coltype | 碰撞类型 0: 正常 1: 有碰撞风险 2: 紧急制动 3: 异常车辆 4.:车辆失控 5: 超载 6: 逆向 7: 变道 8: 他车汇入 9: 他车汇出 10: 本车汇入 11: 本车汇出 12: 向右换道 14: 盲区预警 15: 紧急制动 16: 即将闯红灯 17: 已闯红灯 18: 左转辅助 19: 异常慢行 20: 绿波 21:异常车辆（非 who am i） 22.紧急制动（非 who am i） 23: 弱势交通参与者（非 who am i） |
| distance | 距碰撞物距离（限高/宽/速） |
| orientation | 距碰方向（1:正前方向; 2:正后方向; 3:正右方向; 4:正左方向; 5:右前方向; 6:右后方向: 7:左前方向; 8:左后方向） |

##### 5.2 msg消息结构体类型数据


| **字段**  | **数据类型** | **说明**                                                     | **备注** |
| --------- | ------------ | ------------------------------------------------------------ | -------- |
| Format    | string       | MainCar和OtherCars结构体各字段含义，默认<br />Format="XYZHSTIRDLecdo"<br />X:  相对坐标x，float类型，必要<br />Y：相对坐标y，float类型，必要<br />Z:   相对坐标z，float类型，必要<br />H:   车辆方向角，float类型，必要<br />S:    车辆速度，float类型<br />T： 车辆类型，int类型，具体定义同预警消息中objtype字段，必要<br />I： 车辆Id，string类型，必要<br />R:   道路Id，int类型，必要<br />C:   路段Id，int类型，必要<br />L:    车道Id，int类型，必要<br />e:    预警级别，int类型，可选<br />c:    碰撞类型，int类型，可选<br />d:    碰撞距离，int类型，可选 <br />o:    碰撞方向，int类型，可选 | 必要     |
| MainCar   | object       | 主车信息，格式内容按format规定的顺序解析                     | 必要     |
| OtherCars | []object     | 周车信息，格式内容按format规定的顺序解析                     | 必要     |

`ps: 考虑到周车信息量可能很大，出于效率和性能考虑，mainCar和otherCars结构体省略key信息，采用[]object结构，object为任意数据类型，所以要求mainCar和otherCars结构体中各字段的顺序严格按照format的顺序进行解析。其中e（预警级别）、c（碰撞类型）、d(碰撞距离)、o(碰撞方向) 是可选字段，用来标记与主车存在预警关系的周围车辆。`


##### 5.3 msg结构体参数示例：  
```json
{
    "id": 0,
    "type": 6,
    "around": {
        "Format": "XYZHSTIRCLecdo",
        "MainCar": [-3441.51, -43.67, -2757.93, 5.81, 17.47, 1, "2636", 205003, 3, 3],
        "OtherCars": [[-3492.74, -43.67, -2850.26, 5.81, 24.65, 1, "2806", 205003, 3, 2], [-3532.01, -43.67, -2885.52, 2.67, 28.31, 1, "2206", 227003, 1, 3], [-3453.54, -43.67, -2772.33, 5.8, 25.15, 1, "2721", 205003, 3, 2], [-3454.28, -43.67, -2782.35, 5.77, 18.07, 1, "2710", 205003, 3, 3], [-3532.44, -43.67, -2894.88, 2.66, 24.43, 1, "2078", 227003, 1, 2], [-3422.54, -43.67, -2676.31, 2.67, 16.99, 1, "2116", 227003, 1, 3], [-3479.46, -43.67, -2815.6, 5.81, 35.49, 1, "2778", 205003, 3, 1], [-3449.78, -43.67, -2727.55, 2.68, 18.49, 1, "1745", 227003, 1, 3], [-3388.44, -43.67, -2647.7, 5.78, 19.83, 1, "2679", 205003, 3, 2], [-3419.02, -43.67, -2677.18, 2.66, 25.89, 1, "2931", 227003, 1, 2], [-3416.62, -43.67, -2693.97, 5.82, 33.81, 1, "2892", 205003, 3, 1]],
        "OtherCarsV2": null,
        "Guide": null,
        "Windows": null,
        "TrafficLight": null,
        "ExtraInfo": null,
        "Devices": null,
        "SignalLamp": null,
        "Statistics": null
    }
}
```

`ps: 具体预警组合类型参见附表3、附表4。`


#### 6、IPointEventListener

点事件消息回调接口，需实现函数 `` void onPointEvent(List<PointEventInfo> event);``
可通过``getPointEventDetails``获取详细点事件信息。


----------

### 3.3.5  **ILoginManager 登录及注册相关接口**

1. ``ILoginManager LoginManager.getInstance()``   
    获取ILoginManager接口实例

2. ```Disposable getCode(String phone, LoginListener listener)```
    获取手机验证码。  
    参数phone:11位电话号码;  
    LoginListener:消息返回回调，若获取成功则回调onSuccess，失败回调onErrorMsg

3. ``` Disposable checkCode(String phone, String code, LoginListener<BaseLoginRet<LoginUserData>> listener);```  
    校验验证码。  
    参数phone:11位电话号码;  
    参数code:验证码;  
    listener：消息返回回调，校验失败返回onErrorMsg，校验成功返回onSuccess，同时如果onSuccess中BaseLoginRet对象data.User为空则说明该用户未注册，需进行注册，否则校验登录成功。

4. ``` Disposable checkUserInfo(String openId, String type, LoginListener listener)```  
    微信等三方登录校验。  
    参数openId：三方登录openID,或匿名登录唯一Id。  
    参数type： 登录类型，如"phone"，"weixin"，"anonymous",见 V2xUserConstants.TYPEID_ARRAY。  
    listener： 如果成功返回onSuccess，失败则返回onErrorMsg，若onErrorMsg中msg包含"can`t find user record"则说明该用户未注册，需进行注册，其他错误码则根据提示处理。

5. ```Disposable registerUserInfo(UserInfo userInfo, LoginListener listener)```  
   用户注册。  
   userInfo,用户注册信息，其中以下为必填项：
   Phone/OpenID: 电话号码或者opneId;  
   TypeID:账户类型 (V2xUserConstants.TYPEID_ARRAY)，（"phone"，"weixin"，"alipay"） ；  
   NickName：昵称；  
   CarID：车牌号；  
   CarKind:车种(V2xUserConstants.CARKIND_ARRAY),
   {"社会车辆", "警车", "救护车", "消防车", "工程车"};
   非必填项：
   cartype:车辆类型（V2xUserConstants.CARTYPE_ARRAY），
   {小轿车 SUV MPV 摩托车 巴士 其他车}

6. ``` Disposable updateUserInfo(UserInfo userInfo, LoginListener listener) ```  
    更新完善用户信息。注意若userInfo某个字段为空则会将服务器保存的字段也清空。

7. ``` Disposable updateUserInfo(String id, String typeId, String userInfoParam, LoginListener listener)```  
    更新完善用户信息：  
    参数id : 如果是手机号登录则是phone,微信登录则是openId。(userInfo.Phone，userInfo.OpenID)
    参数typeId ：账户类型id(userInfo.TypeID ，"phone"，"weixin"，"alipay")
    参数userInfoParam：需要修改的参数序列化后的内容，可根据函数```createUpdateUserParam```来生成。  

8. ```String createUpdateUserParam(String AvatarUrl, String NickName, String Gender, String IDNumber)```  
    创建更新用户某个信息的参数，结合第6点updateUserInfo函数使用。  

9. ``` Disposable deleteUserInfo(UserInfo userInfo, LoginListener listener) ```  
    账户注销。
    其中userInfo中一下参数必须：
    Phone/OpenID: 电话号码或者opneId;  
    TypeID：账户类型。

10. ```boolean isUserLogin()```  
    根据本地缓存判断是否有用户已经登录过。

11. ``` void cleanLocalLoginInfo()```  
    清除本地用户登录信息,并断开当前用户连接信息。

12. ``` String getLocalLoginOpenId()```  
    获取本地登录的账户，如果是手机号登录的则返回手机号，否则返回openId，如果没有登录则返回null。
    
13. ```
    String getUserLoginType() 
    ```

​		获取登录类型，手机号/微信/匿名

14. ```
    boolean isAnonymousLogin() 
    ```

​		判断是否是匿名登录



### 3.3.6 其他网络相关接口 INetManager

1. ```Disposable submitTicket(String mask, LoginListener listener);```

   提交工单系统，上报错误信息。

   参数：

   mask： 错误简介，例如"前向碰撞延迟"等信息。

   listener：上报结果回调。

2. ``` Disposable getPointEventDetails(String id, LoginListener listener) ```

   根据点事件id,获取事件详情。

### 3.3.7 微信登录接口WxLoginModel

1、`static WxLoginModel getInstance()` 

获取微信登录实例

2、`int startWxAuthReq(Context context, String appId, String appSec)`

开始微信授权登录请求

3、`setWxLoginCallBack(IWxLoginCallBack callBack)`

设置微信登录回调。

### 3.3.8 **ColNotifyManager 预警通知管理类** 

​			预警管理是sdk内部对收到的预警做了一定的处理，给出了一种用户提示的解决方案。根据设置的预警频率，在该时间段内最多发送一次预警，内容包含是否弹窗提醒、是否需要播报语音、播报内容、预警等级、预警类型等。

1. ``ColNotifyManager getInstance()``  
   获取预警通知管理类。
2. ``void setNotifyEventInterval(int millis)``

​		设置预警通知频率,单位ms,默认1000ms。

3. ``void setOnColNotifyListener(IColNotifyListener colNotifyListener)``    
   设置预警通知回调。   
   ``IColNotifyListener``包含一个`` void onColNotify(ColNotifyInfo colNotifyInfo)``函数。  
   ``ColNotifyInfo``参数：    
   text：String，语音或文字播报内容；   
   type： int，预警类型；   
   level：int 预警等级；   
   needSpeech: boolean,是否需要语音播报;   
   needAlert: boolean,是否需要弹窗；    

4. ``void release()``   
当不需要监听预警通知时，调用此函数释放资源。


### 3.3.9 日志导出, LogExportUtils类 
由于android分区存储，导致手机文件管理无法直接查看日志，通过日志导出功能可以将sdk日志导出至系统download目录，从而方便测试人员分析问题。建议在开发调试阶段开放。  

1. ``void exportCurLog(Activity activity,ILogExportListener listener)``

   导出日志操作，用户触发导出日志事件时调用。

    参数：

    listener:监听日志导出结果，为null则不关心导出结果。

2. ``void onExportCurLog(Activity activity,int requestCode,int resultCode,Intent data,vararg String paths)``  

   参数paths: 可变参数，可以传入额外需要一同打包进zip的路径。默认已经包含返回sdk全部日志。

   在Activity的``onActivityResult``回调中调用此方法，参数即``onActivityResult``回调参数。

### 3.3.10 ObuConfig2，obu属性配置

需添加依赖：``implementation "com.moandjiezana.toml:toml4j:$rootProject.ext.deps.toml4j"``

改配置用于在obu模式下设置obu相关属性。设置参数完毕后，如果需要立即生效需调用`` ObuConfigurer.enableConfigs()``

1. ```java
   public static void setCurMode(Context context, String mode)
   ```

    设置当前路段，目前支持成宜、二绕

    mode常量：

   ```java
   ObuConfig2.VAL_MODE_CHENGYI //成宜
   ObuConfig2.VAL_MODE_ERRAO //二绕
   ```
   
2. ```java
   public static String getCurMode(Context context)
   ```

   获取当前路段值。

3. ```java
   public static ObuConfig2 Builder(@NonNull Context context) 
   ```

   获取ObuConfig2对象，用于获取参数或者设置参数。  
   
4. ```java
   public <T> ObuConfig2 setConfigVal(@NonNull String key, T val)
   ```

​		修改当前mode下对应key的值。

5. ```java
   public <T> T getConfVal(String key, @NonNull T defVal)
   ```

   获取当前mode下的对应key的值。

6. ```java
   public void build()
   ```

   保存当前设置的值。



### 3.3.11 ObuConfigurer,obu配置使能类

1. ```java
   static void enableConfigs(Context context)
   ```

   使能配置立即生效。

   每当通过setConfigVal设置属性时，该次设置的值会保存，下次app重新启动生效。如果需当前立即生效，请在设置完毕后调用此函数。





# 附录 

## 附表1. 城市场景预警匹配表

| **场景**             | **coltype** | **objtype** | **flag** | **方向** |
|--------------------| ----------- | ----------- | -------- |--------|
| 前向碰撞               |             |             |          |        |
| 普通车                | 1           | 1           | 0        | 1      |
| 摩托车                | 1           | 13          | 0        | 1      |
| 卡车/大型汽车            | 1           | 4           | 0        | 1      |
| 后向碰撞               |             |             |          |        |
| 普通车                | 1           | 1           | 0        | 2      |
| 摩托车                | 1           | 13          | 0        | 2      |
| 卡车/大型汽车            | 1           | 4           | 0        | 2      |
| 弱势交通参与者            | -           | -           | -        | -      |
| 动物预警               | 1           | 3           | 0        | 1、5、7  |
| 行人预警               | 1           | 2           | 0        | 1、5、7  |
| 鬼探头预警              | 1           | 2           | 2        | 5、7    |
| 非机动车预警             | 1           | 12          | 0        | 1、5、7  |
| 后向碰撞               | 25          | 1           | 0        | 2      |
| 弱势交通参与者(非who am i) |             |             |          |        |
| 动物预警               | 23          | 3           | 0        | 1      |
| 行人预警               | 23          | 2           | 0        | 1      |
| 鬼探头预警              | 23          | 2           | 2        | 1      |
| 紧急制动预警             |             |             |          |        |
| 普通车                | 2           | 1           | 0        | 1      |
| 卡车/大型汽车            | 2           | 4           | 0        | 1      |
| 紧急制动预警(非who am i)  | 22          | 1           | 0        | 1      |
| 车辆失控预警             |             |             |          |        |
| 小型车辆失控预警           | 4           | 1           | 0        | 1、5、7  |
| 大型车辆失控预警           | 4           | 4           | 0        | 1、5、7  |
| 异常车辆提醒             | -           | -           | -        | -      |
| 异常慢行预警             | 19          | 1           | 0        | 1、5、7  |
| 异常停车预警             | 3           | 1           | 0        | 1、5、7  |
| 异常车辆提醒(非whoami)    |             |             |          |        |
| 异常停车预警             | 21          | 1           | 1        | 1      |
| 闯红灯预警              | -           | -           | -        | -      |
| 即将闯红灯预警            | 16          | 32          | 0        | 1      |
| 已闯红灯警告             | 17          | 32          | 0        | 1      |
| 交叉路口碰撞预警           |             |             |          |        |
| 交叉路口碰撞预警           | 8           | 1           | 0        | 5、7    |
| 交叉路口双向来车预警         | 8           | 1           | 0        | 9      |
| 左转辅助               | 18          | 1           | 0        | 1      |
| 盲区预警/变道辅助          | 14          | 1           | 0        | 6、8    |
| 逆向超车预警             | 6           | 1           | 0        | 1      |
| 道路危险状况提示           | -           | -           | -        | -      |
| 积水预警               | 0           | 21          | 0        | 1、5、7  |
| 道路施工               | 0           | 10          | 0        | 1      |
| 深坑预警               | 0           | 20          | 0        | 1、5、7  |
| 湿滑预警               | 0           | 25          | 0        | 1、5、7  |
| 急转弯预警              | 0           | 26          | 0        | 1      |
| 交通事故               | 0           | 9           | 0        | 1      |
| 交通管制               | 0           | 30          | 0        | 1      |
| 交通拥堵               | 0           | 15          | 0        | 1      |
| 道路变窄               | 0           | 43          | 0        | 1      |
| 落石                 | 0           | 6           | 0        | 1      |
| 结冰                 | 0           | 44          | 0        | 1      |
| 抛洒物                | 0           | 11          | 0        | 1      |
| 大雨                 | 0           | 17          | 0        | 1      |
| 团雾                 | 0           | 18          | 0        | 1      |
| 紧急车辆提醒             | -           | -           | -        | -      |
| 警车预警               | 0           | 5           | 0        | 2、6、8  |
| 消防车预警              | 0           | 19          | 0        | 2、6、8  |
| 救护车预警              | 0           | 7           | 0        | 2、6、8  |
| 前方拥堵提醒             | 0           | 15          | 0        | 1、5、7  |
| 车内标牌               | -           | -           | -        | -      |
| 限高预警               | 0           | 23          | 0        | 1      |
| 限宽预警               | 0           | 24          | 0        | 1      |
| 限重预警               | 0           | 33          | 0        | 1      |
| 限速预警               | 0           | 22          | 0        | 1      |
| 解除限速预警             | 0           | 31          | 0        | 1      |
| 禁止左转/右转/直行         | 0           | 34/35/36    | 0        | 1      |
| 禁止掉头               | 0           | 37          | 0        | 1      |
| 禁止超车               | 0           | 38          | 0        | 1      |
| 禁止停车               | 0           | 39          | 0        | 1      |
| 前方学校               | 0           | 41          | 0        | 1      |
| 前方事故多发             | 0           | 42          | 0        | 1      |
| 禁止鸣笛               | 0           | 40          | 0        | 1      |
| 绿波车速引导             | 20          | 32          | 0        | 1      |


## 附表2. 高速场景预警匹配表

| **场景**     | **coltype** | **objtype** | **flag** | **方向** |
|------------| ----------- | ----------- | -------- |--------|
| 前向碰撞       |             |             |          |        |
| 小型汽车       | 33          | 1           | 0        | 1      |
| 大型汽车       | 33          | 4           | 0        | 1      |
| 后向碰撞       |             |             |          |        |
| 小型汽车       | 33          | 1           | 0        | 2      |
| 大型汽车       | 33          | 4           | 0        | 2      |
| 紧急制动       |             |             |          |        |
| 小型汽车       | 34          | 1           | 0        | 1      |
| 大型汽车       | 34          | 4           | 0        | 1      |
| 盲区预警       | 35          | 1           | 0        | 5、7    |
| 失控预警       |             |             |          |        |
| 小型汽车       | 30          | 1           | 0        | 1      |
| 大型汽车       | 30          | 4           | 0        | 1      |
| 异常停车       | -           | -           | -        | -      |
| 小型汽车       | 31          | 1           | 0        | 1、5、7  |
| 异常慢行       |             |             |          |        |
| 小型汽车       | 32          | 1           | 0        | 1、5、7  |
| 道路危险状况提示   | -           | -           | -        | -      |
| 积水预警       | 0           | 45          | 0        | 1、5、7  |
| 道路施工       | 0           | 46          | 0        | 1      |
| 深坑预警       | 0           | 47          | 0        | 1、5、7  |
| 湿滑预警       | 0           | 48          | 0        | 1、5、7  |
| 急转弯预警      | 0           | 49          | 0        | 1      |
| 交通事故       | 0           | 50          | 0        | 1      |
| 交通管制       | 0           | 51          | 0        | 1      |
| 交通拥堵       | 0           | 52          | 0        | 1      |
| 道路变窄       | 0           | 53          | 0        | 1      |
| 落石         | 0           | 54          | 0        | 1      |
| 结冰         | 0           | 55          | 0        | 1      |
| 抛洒物        | 0           | 56          | 0        | 1      |
| 大雨         | 0           | 57          | 0        | 1      |
| 团雾         | 0           | 58          | 0        | 1      |
| 前方拥堵提醒     | 0           | 52          | 0        | 1、5、7  |
| 车内标牌       | -           | -           | -        | -      |
| 限高预警       | 0           | 59          | 0        | 1      |
| 限宽预警       | 0           | 60          | 0        | 1      |
| 限重预警       | 0           | 61          | 0        | 1      |
| 限速预警       | 0           | 62          | 0        | 1      |
| 解除限速预警     | 0           | 63          | 0        | 1      |
| 禁止左转/右转/直行 | 0           | 64/65/66    | 0        | 1      |
| 禁止掉头       | 0           | 67          | 0        | 1      |
| 禁止超车       | 0           | 68          | 0        | 1      |
| 禁止停车       | 0           | 69          | 0        | 1      |
| 前方学校       | 0           | 71          | 0        | 1      |
| 前方事故多发     | 0           | 72          | 0        | 1      |
| 禁止鸣笛       | 0           | 70          | 0        | 1      |
| 前方服务区      | 0           | 73          | 0        | 1      |

## 附表3. 周车城市场景预警匹配表

| **场景**                   | **coltype** | **objtype** |
| -------------------------- | ----------- | ----------- |
| 前向碰撞                   |             |             |
| 普通车                     | 1           | 1           |
| 摩托车                     | 1           | 13          |
| 卡车/大型汽车              | 1           | 4           |
| 弱势交通参与者             | -           | -           |
| 动物预警                   | 1           | 3           |
| 行人预警                   | 1           | 2           |
| 非机动车预警               | 1           | 12          |
| 弱势交通参与者(非who am i) |             |             |
| 动物预警                   | 23          | 3           |
| 行人预警                   | 23          | 2           |
| 鬼探头预警                 | 23          | 2           |
| 紧急制动预警               |             |             |
| 普通车                     | 2           | 1           |
| 卡车/大型汽车              | 2           | 4           |
| 紧急制动预警(非who am i)   | 22          | 1           |
| 车辆失控预警               |             |             |
| 小型车辆失控预警           | 4           | 1           |
| 大型车辆失控预警           | 4           | 4           |
| 异常车辆提醒               |             |             |
| 异常慢行预警               | 19          | 1           |
| 异常停车预警               | 3           | 1           |
| 异常车辆提醒(非whoami)     |             |             |
| 异常停车预警               | 21          | 1           |
| 闯红灯预警                 |             |             |
| 即将闯红灯预警             | 16          | 32          |
| 已闯红灯警告               | 17          | 32          |
| 交叉路口碰撞预警           |             |             |
| 交叉路口碰撞预警           | 8           | 1           |
| 左转辅助                   | 18          | 1           |
| 盲区预警/变道辅助          | 14          | 1           |
| 逆向超车预警               | 6           | 1           |
| 道路危险状况提示           |             |             |
| 积水预警                   | 0           | 21          |
| 道路施工                   | 0           | 10          |
| 深坑预警                   | 0           | 20          |
| 湿滑预警                   | 0           | 25          |
| 急转弯预警                 | 0           | 26          |
| 交通事故                   | 0           | 9           |
| 交通管制                   | 0           | 30          |
| 交通拥堵                   | 0           | 15          |
| 道路变窄                   | 0           | 43          |
| 落石                       | 0           | 6           |
| 结冰                       | 0           | 44          |
| 抛洒物                     | 0           | 11          |
| 大雨                       | 0           | 17          |
| 团雾                       | 0           | 18          |
| 紧急车辆提醒               |             |             |
| 警车预警                   | 0           | 5           |
| 消防车预警                 | 0           | 19          |
| 救护车预警                 | 0           | 7           |
| 前方拥堵提醒               | 0           | 15          |
| 车内标牌                   |             |             |
| 限高预警                   | 0           | 23          |
| 限宽预警                   | 0           | 24          |
| 限重预警                   | 0           | 33          |
| 限速预警                   | 0           | 22          |
| 解除限速预警               | 0           | 31          |
| 禁止左转/右转/直行         | 0           | 34/35/36    |
| 禁止掉头                   | 0           | 37          |
| 禁止超车                   | 0           | 38          |
| 禁止停车                   | 0           | 39          |
| 前方学校                   | 0           | 41          |
| 前方事故多发               | 0           | 42          |
| 禁止鸣笛                   | 0           | 40          |
| 绿波车速引导               | 20          | 32          |

## 附表4. 周车高速场景预警匹配表

| **场景**           | **coltype** | **objtype** |
| ------------------ | ----------- | ----------- |
| 前向碰撞           |             |             |
| 小型汽车           | 33          | 1           |
| 大型汽车           | 33          | 4           |
| 紧急制动           |             |             |
| 小型汽车           | 34          | 1           |
| 大型汽车           | 34          | 4           |
| 盲区预警           | 35          | 1           |
| 失控预警           |             |             |
| 小型汽车           | 30          | 1           |
| 大型汽车           | 30          | 4           |
| 异常停车           |             |             |
| 小型汽车           | 31          | 1           |
| 异常慢行           |             |             |
| 小型汽车           | 32          | 1           |
| 道路危险状况提示   |             |             |
| 积水预警           | 0           | 45          |
| 道路施工           | 0           | 46          |
| 深坑预警           | 0           | 47          |
| 湿滑预警           | 0           | 48          |
| 急转弯预警         | 0           | 49          |
| 交通事故           | 0           | 50          |
| 交通管制           | 0           | 51          |
| 交通拥堵           | 0           | 52          |
| 道路变窄           | 0           | 53          |
| 落石               | 0           | 54          |
| 结冰               | 0           | 55          |
| 抛洒物             | 0           | 56          |
| 大雨               | 0           | 57          |
| 团雾               | 0           | 58          |
| 前方拥堵提醒       | 0           | 52          |
| 车内标牌           | -           | -           |
| 限高预警           | 0           | 59          |
| 限宽预警           | 0           | 60          |
| 限重预警           | 0           | 61          |
| 限速预警           | 0           | 62          |
| 解除限速预警       | 0           | 63          |
| 禁止左转/右转/直行 | 0           | 64/65/66    |
| 禁止掉头           | 0           | 67          |
| 禁止超车           | 0           | 68          |
| 禁止停车           | 0           | 69          |
| 前方学校           | 0           | 71          |
| 前方事故多发       | 0           | 72          |
| 禁止鸣笛           | 0           | 70          |
| 前方服务区         | 0           | 73          |

 

