# 前言　

在iOS8里，苹果开放了NetworkExtesion,允许App创建VPN配置，使用系统内置的VPN协议(IPSec,IKEv2). 

在iOS9里，苹果在NetworkExtesion中开放了一个极为强大的接口，允许App 创建一块Tun网卡和路由表来管理所有的网络流量。

但是从iOS4.1开始，就有一些厂商的app，就可以创建SSL VPN，这是一个真正全局的VPN。这是怎么回事呢。

这篇博客里有提及到：[iOS 4.1: Undocumented VPN API, used by Cisco AnyConnect](https://blog.michael.kuron-germany.de/2010/09/ios-4-1-undocumented-vpn-api-used-by-cisco-anyconnect/)

苹果给这些大厂商进行了授权，准允他们使用私人的接口。创建Tun虚拟接口，取到一个原始的Socket FD,通过此FD来读写数据到Tun口

使用Hopper打开iOS dy_shared_cache里的 SystemConfiguration可以看到这份厂商名单

## 关于这个项目

逆向接口拿到Socket FD,因为大部分代理相关的功能都可以使用NEKit来完成。
所以通过我们写的一层中间接口从Socket FD,创建一个NEPacketTunnelFlow的内部实现NEVirtualInterface。

从而与iOS9的读写Tun口数据的两个高级接口兼容。
[NEPacketTunnelFlow readPackets]　
[NEPacketTunnelFlow readMultiPacketsWithCompletionHandler]

这样就可以把原来在PacketTunnel中的实现，一层不变的移植到iOS8上.

### 几个关键的点
1. 首先确保项目的配置文件里置有这个字段，有了这个，在installd安装app的时候才会把搜索你的安装包，把包里后缀名为.vpnplugin的文件夹拷贝到特定目录，供neagent加载。 .vpnplugin里的文件可以按照Framework的结构来生成. 

   ```
	<key>UIVPNPlugin</key>
	<array>
		<string>com.cisco.anyconnect.applevpn.plugin</string>
	</array>
	```

2. 授权证书, 之前说过，必须有大厂商的授权才能使用这些私有接口.
    这里我们就使用cisco anyconnect的授权
    
    ```
    <key>com.apple.networking.vpn.configuration</key>
    <array>
        <string>com.cisco.anyconnect.applevpn.plugin</string>
    </array>
    ```

由于我们没有这个授权对应的开发者账号，所以在xcode对app进行签名的时候是肯定会出错的。　所以我们还要在Xcode上作一些手脚，让其跳过签名，生成不包含签名的app.　

关于如何禁用Xcode Code Sign，Google 上有一大批文章。
关键是一个叫SDKSettings.plist文件里的相应字段.

3. 修改AppSync，让iOS顺利安装我们没有签名的ipa.
    上面说过了.vpnplugin里的dylib由neagent来加载。跟installd一样，neagent加载资源时也会验证资源的签名。所以我们可以修改AppSync让neagent加载未签名的文件。
    
4. 自已在iOS8里实现iOS9里的高级接口 

 ```
-[NEPacketTunnelFlow readMultiPacketsWithCompletionHandler:]
-[NEPacketTunnelFlow writePackets:withProtocols:]
```

这里的关键是从hopper里逆向出的一组低级函数: 

```
typedef CFTypeRef NEVirtualInterface;

//创建
NEVirtualInterface NEVirtualInterfaceCreateFromSocket(CFAllocatorRef allocator, int fd, dispatch_queue_t queue, int flag);

//设置为自动读出数据
NEVirtualInterface NEVirtualInterfaceSetReadAutomatically(NEVirtualInterface arg0, int flag );

//设置数据回调接口
typedef void(^MyBlock5Old)(NEVirtualInterface vi,int protocol,Byte* buffer,int length);
void NEVirtualInterfaceSetReadIPPacketHandler(NEVirtualInterface vi, MyBlock5Old callback );//ios8

//设置Interface的状态为可以开始读数据了,arg2 BOOL
NEVirtualInterface NEVirtualInterfaceReadyToRead( NEVirtualInterface vi,  int flag);

//写入数据
void NEVirtualInterfaceWriteIPPacket(NEVirtualInterface vi,int protocol,Byte* buffer,int length);

```

5. 关于从Socket FD里读到的数据格式

每一组数据前面8个字节是协议的类型。用于标记当前是IPV4 或 IPV6,后面是IP Packet


## 成品

Cydia源: http://23.105.215.216/cydia
手机隧道 for iOS8

## Test

Test in iOS8


## References

[iOS Tun 网卡数据的流动](https://blog.csdn.net/naipeng/article/details/54972404)


