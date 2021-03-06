# 开发指南\(Java\) {#concept_cgg_k2v_y2b .concept}

本文档介绍如何配置服务端订阅、接入HTTP/2 Java版SDK、进行身份认证和设置消息接收接口。

SDK配置方法：

-   有关设备端接入配置，请下载[iot-http2-sdk-demo](http://aliyun-iot.oss-cn-hangzhou.aliyuncs.com/java-http2-sdk-demo/iot-http2-sdk-demos.zip)，并参考[配置HTTP/2 Java SDK](../../../../../intl.zh-CN/设备端开发指南/C-SDK/配置HTTP__2 Java SDK.md#)接入物联网平台。
-   有关服务端订阅配置，请下载[HTTP/2 Java SDK demo](http://aliyun-iot.oss-cn-hangzhou.aliyuncs.com/java-http2-sdk-demo/http2-server-side-demo.zip)，参考本文以下章节进行配置。

以下简单介绍服务端订阅的开发流程。

## 配置服务端订阅 {#section_tbd_2s5_42b .section}

1.  登录[物联网平台控制台](https://iot.console.aliyun.com/product/region/cn-shanghai)。
2.  左侧导航栏选择**设备管理** \> **产品**。
3.  在产品列表中，搜索到要配置服务端订阅的产品，并单击该产品对应的**查看**按钮，进入产品详情页。
4.  单击**服务端订阅** \> **设置**。
5.  选择推送的消息类型。

    ![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/18850/154833572012666_zh-CN.png)

    -   设备上报消息：指产品下所有设备 Topic 列表中，具有发布权限的 Topic 中的消息。勾选后，可以通过 HTTP/2 SDK 接收这些消息。

        高级版产品的设备上报消息，包括设备上报的自定义数据和属性、事件、属性设置响应、服务调用响应的物模型数据。而基础版产品只包括设备上报的自定义数据。

        例如，有一个高级版产品，有3个Topic类，分别是：

        -   `/${YourProductKey}/${YourDeviceName}/user/get`，具有订阅权限。
        -   `/${YourProductKey}/${YourDeviceName}/user/update`，具有发布权限。
        -   `/sys/${YourProductKey}/${YourDeviceName}/thing/event/property/post`，具有发布权限。
        那么，服务端订阅会推送具有发布权限的Topic类中的消息，即`/${YourProductKey}/${YourDeviceName}/user/update`和`/sys/${YourProductKey}/${YourDeviceName}/thing/event/property/post`中的消息。其中，`/sys/${YourProductKey}/${YourDeviceName}/thing/event/property/post`中的数据已经过系统处理。

    -   设备状态变化通知：指一旦该产品下的设备状态变化时通知的消息，例如设备上线、下线的消息。设备状态消息的发送 Topic 为 `/as/mqtt/status/${YourProductKey}/${YourDeviceName}`。勾选后，可以通过 HTTP/2 SDK 接收设备状态变化的通知消息。
    -   网关发现子设备上报：高级版产品特有的消息类型，网关可以将发现的子设备上报，需要网关上的应用程序支持。
    -   设备拓扑关系变更：高级版产品特有的消息类型，指的是子设备和网关之间的拓扑关系建立和解除。
    -   设备生命周期变更：高级版产品特有的消息类型，包括设备创建、删除、禁用、启用等消息的订阅。

## 接入 SDK {#section_v3d_gj5_42b .section}

在工程中添加 maven 依赖接入 SDK。

```
<dependency>
    <groupId>com.aliyun.openservices</groupId>
    <artifactId>iot-client-message</artifactId>
    <version>1.1.3</version>
</dependency>

<dependency>
    <groupId>com.aliyun</groupId>
    <artifactId>aliyun-java-sdk-core</artifactId>
    <version>3.7.1</version>
</dependency>
```

## 身份认证 {#section_atv_yl5_42b .section}

使用服务端订阅功能，需要基于您的阿里云 AccessKey 进行身份认证并建立连接。

建立链接示例如下：

```
// 阿里云accessKey
        String accessKey = "xxxxxxxxxxxxxxx";
        // 阿里云accessSecret
        String accessSecret = "xxxxxxxxxxxxxxx";
        // regionId
        String regionId = "cn-shanghai";
        // 阿里云uid
        String uid = "xxxxxxxxxxxx";
        // endPoint:  https://${uid}.iot-as-http2.${region}.aliyuncs.com
        String endPoint = "https://" + uid + ".iot-as-http2." + regionId + ".aliyuncs.com";

        // 连接配置
        Profile profile = Profile.getAccessKeyProfile(endPoint, regionId, accessKey, accessSecret);

        // 构造客户端
        MessageClient client = MessageClientFactory.messageClient(profile);

        // 数据接收
        client.connect(messageToken -> {
            Message m = messageToken.getMessage();
            System.out.println("receive message from " + m);
            return MessageCallback.Action.CommitSuccess;
        });
```

accessKey 即您的账号的 AccessKey ID， accessSecret 即 AccessKey ID对应的 AccessKey Secret。请登录[阿里云控制台](https://home.console.aliyun.com/new#/)，将光标移至您的账号头像上，在选项框中选择 **accesskeys** 查看您的 AccessKey ID 和 AccessKey Secret；选择**安全设置**查看您的账号 ID。

regionId为您的物联网平台服务地域。

## 设置消息接收接口 {#section_fnd_3x5_42b .section}

连接建立后，服务端会立即向 SDK 推送已订阅的消息。因此，建立链接时，需要提供消息接收接口，用于处理未设置回调的消息。建议在`connect`之前，调用 setMessageListener 设置消息回调。

您需要通过 MessageCallback接口的`consume`方法，和调用 `messageClient` 的`setMessageListener()`方法来设置消息接收接口。

`consume` 方法的返回值决定 SDK 是否发送 ACK。

设置消息接收接口的方法如下：

```
MessageCallback messageCallback = new MessageCallback() {
    @Override
    public Action consume(MessageToken messageToken) {
        Message m = messageToken.getMessage();
        log.info("receive : " + new String(messageToken.getMessage().getPayload()));
        return MessageCallback.Action.CommitSuccess;
    }
};
messageClient.setMessageListener("/${YourProductKey}/#",messageCallback);
```

其中，

-   参数 MessageToken 指消息回执的消息体。通过`MessageToken.getMessage()`可获取消息体。MessageToken可以用于手动回复 ACK 。

    消息体包含的内容如下：

    ```
    public class Message {
        // 消息体
        private byte[] payload;
        // Topic 
        private String topic;
        // 消息ID
        private String messageId;
        // QoS
        private int qos;
    }
    ```

-   具体请参考[消息体格式](#)。
-   `messageClient.setMessageListener("/${YourProductKey}/#",messageCallback);`用于设置回调。本示例中，设置为指定 Topic 回调。

    您可以设置为指定 Topic 回调，也可以设置为通用回调。

    -   指定 Topic 回调

        指定 Topic 回调的优先级高于通用回调。一条消息匹配到多个 Topic 时，按字典顺序优先调用，并且仅回调一次。

        设置回调时，可以指定带通配符的 Topic，如 `/${YourProductKey}/${YourDeviceName}/#`。

        示例：

        ```
        messageClient.setMessageListener("/alEddfaXXXX/device1/#",messageCallback);
        //当收到消息的Topic，如"/alEddfaXXXX/device1/update"，匹配指定Topic时，会优先调用该回调
        ```

    -   通用回调

        未指定 Topic 回调的消息，则调用通用回调。

        设置通用回调方法：

        ```
        messageClient.setMessageListener(messageCallback);
        //当收到消息topic未匹配到已定指的Topic 回调时，调用该回调
        ```

-   设置回复 ACK。

    对于 QOS\>0 的消息，消费后需要回复 ACK。SDK 支持自动回复 ACK 和手动回复 ACK。默认为自动回复 ACK。本示例中未设置回复 ACK，则默认为自动回复。

    -   自动回复 ACK：设置为自动回复 ACK 后，若 `MessageCallback.consume` 的返回值为 true 则 SDK 会发送 ACK；返回 false 或抛出异常，则不会返回 ACK。对于 QOS\>0 且未回复 ACK 的消息，服务端会重新发送。
    -   手动回复 ACK：通过`MessageClient.setManualAcks`设置手动回复 ACK。

        设置为手动回复 ACK 后，需要调用`MessageClient.ack()`方法回复 ACK，参数为 MessageToken。MessageToken 参数值可在接收消息中获取。

        手动回复 ACK 的方法：

        ```
        messageClient.ack(messageToken);
        ```


## 消息体格式 {#section_csq_yv5_4fb .section}

-   设备状态通知：

    ```
    {
        "status":"online|offline",
        "productKey":"12345565569",
        "deviceName":"deviceName1234",
        "time":"2018-08-31 15:32:28.205",
        "utcTime":"2018-08-31T07:32:28.205Z",
        "lastTime":"2018-08-31 15:32:28.195",
        "utcLastTime":"2018-08-31T07:32:28.195Z",
        "clientIp":"123.123.123.123"
    }
    ```

    |参数|类型|说明|
    |--|--|--|
    |status|String|设备状态，online上线，offline离线|
    |productKey|String|设备所属产品的唯一标识|
    |deviceName|String|设备名称|
    |time|String|发送通知的时间点|
    |utcTime|String|发送通知的UTC时间点|
    |lastTime|String|状态变更时最后一次通信时间|
    |utcLastTime|String|状态变更时最后一次通信的UTC时间|
    |clientIp|String|设备公网出口IP|

    **说明：** 为避免消息时序紊乱造成影响，建议您根据lastTime来维护最终设备状态。

-   设备生命周期变更：

    ```
    {
        "action" : "create|delete|enable|disable",
        "iotId" : "4z819VQHk6VSLmmBJfrf00107ee201",
        "productKey" : "12345565569",
        "deviceName" : "deviceName1234",
        "deviceSecret" : "",
        "messageCreateTime": 1510292739881
    }
    ```

    |参数|类型|描述|
    |--|--|--|
    |action|String|     -   create：创建设备
    -   delete：删除设备
    -   enable：启用设备
    -   disable：禁用设备
 |
    |iotId|String|设备在平台内的唯一标识|
    |productKey|String|设备所属产品的ProductKey|
    |deviceName|String|设备名称|
    |deviceSecret|String|设备密钥，仅在action为create时包含该参数|
    |messageCreateTime|Long|消息产生的时间戳，单位为毫秒|

-   设备拓扑关系变更：

    ```
    {
        "action" : "add|remove|enable|disable",
        "gwIotId": "4z819VQHk6VSLmmBJfrf00107ee200",
        "gwProductKey": "1234556554",
        "gwDeviceName": "deviceName1234",
        "devices": [
            {
              "iotId": "4z819VQHk6VSLmmBJfrf00107ee201",
              "productKey": "12345565569",
              "deviceName": "deviceName1234"
           }
        ],
        "messageCreateTime": 1510292739881
    }
    ```

    |参数|类型|说明|
    |--|--|--|
    |action|String|     -   add：新增拓扑关系
    -   remove：移除拓扑关系
    -   enable：启用拓扑关系
    -   disable：禁用拓扑关系
 |
    |gwIotId|String|网关设备在平台内的唯一标识|
    |gwProductKey|String|网关设备所属产品的ProductKey|
    |gwDeviceName|String|网关设备名称|
    |devices|Object|变更的子设备列表|
    |iotId|String|子设备在平台内的唯一标识|
    |productKey|String|子设备所属产品的ProductKey|
    |deviceName|String|子设备名称|
    |messageCreateTime|Long|消息产生的时间戳，单位为毫秒|

-   网关发现子设备上报：

    ```
    {
        "gwIotId":"4z819VQHk6VSLmmBJfrf00107ee200",
        "gwProductKey":"1234556554",
        "gwDeviceName":"deviceName1234",
        "devices":[
            {
                "iotId":"4z819VQHk6VSLmmBJfrf00107ee201",
                "productKey":"12345565569",
                "deviceName":"deviceName1234"
            }
        ]
    }
    ```

    |参数|类型|说明|
    |--|--|--|
    |gwIotId|String|网关设备在平台内的唯一标识|
    |gwProductKey|String|网关产品的唯一标识|
    |gwDeviceName|String|网关设备名称|
    |devices|Object|发现的子设备列表|
    |iotId|String|子设备在平台内的唯一标识|
    |productKey|String|子设备产品的唯一标识|
    |deviceName|String|子设备名称|


