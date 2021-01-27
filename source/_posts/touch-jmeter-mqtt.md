---
title: jmeter-mqtt基础使用
date: 2019-04-23 18:24:14
tags:
  - jmeter
  - Matt
---

## jmeter-mqtt使用

### 安装

mqtt的测试需要添加一个插件，下载地址在这<https://github.com/emqx/mqtt-jmeter>

安装也很简单，将下载好的文件放到`$JMETER_HOME/lib/ext`下即可，重启之后即可看到如图红色框框中的内容

![image](https://ws2.sinaimg.cn/large/7fa15162gy1g2cmbe3x61j20n40ej42e.jpg)



### 使用

里面一些配置项说明直接复制过来了，下面的小标题对应着gui中各个位置的说明

MQTT Connection

> This section includes basic connection settings.
>
> - **Server name or IP**: The MQTT target to be tested. It can be either IP address or server name. The default value is 127.0.0.1. **DO NOT** add protocol (e.g. tcp:// or ssl://) before server name or IP address!
> - **Port number**: The port opened by MQTT server. Typically 1883 is for TCP protocol, and 8883 for SSL protocol.
> - **MQTT version**: The MQTT version, default is 3.1, and another option is 3.1.1. Sometimes we found version 3.1.1 is required to establish connection to [Azure IoTHub](https://github.com/emqtt/mqtt-jmeter/issues/21).
> - **Timeout(s)**: The connection timeout seconds while connecting to MQTT server. The default is 10 seconds.

MQTT Protocol

> The sampler supports 2 protocols, TCP and SSL. For SSL protocol, it includes normal SSL and dual SSL authentication.
>
> If **'Dual SSL authentication'** is checked, please follow 'Certification files for SSL/TLS connections' at end of this doc to set the client SSL configuration properly.

![image](https://ws3.sinaimg.cn/mw690/7fa15162gy1g2cpectvl2j20v603ymxb.jpg)

User authentication

> User can configure MQTT server with user name & password authentication, refer to [EMQ user name and password authentication guide](http://emqtt.com/docs/v2/guide.html#id3).
>
> - **User name**: If MQTT server is configured with user name, then specify user name here.
> - **Password**: If MQTT server is configured with password, then specify password here.

Connection options

> - **ClientId**: Identification of the client, i.e. virtual user or JMeter thread. Default value is 'conn_'. If 'Add random client id suffix' is selected, JMeter plugin will append generated uuid as suffix to represent the client, otherwise, the text of 'ClientId' will be passed as 'clientId' of current connection.
> - **Keep alive(s)**: Ping packet send interval in seconds. Default value is 300, which means each connection sends a ping packet to MQTT server every 5 minutes.
> - **Connect attampt max**: The maximum number of reconnect attempts before an error is reported back to the client on the first attempt by the client to connect to a server. Set to -1 to use unlimited attempts. Defaults to 0.
> - **Reconnect attampt max**: The maximum number of reconnect attempts before an error is reported back to the client after a server connection had previously been established. Set to -1 to use unlimited attempts. Defaults to 0.
> - **Clean session**: If you want to maintain state information between sessions, set it to false; otherwise, set it to true.



### connect



