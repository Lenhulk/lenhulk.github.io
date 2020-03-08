---
title: iOS应用内检测网络连接是否开启代理(VPN)
date: 2019-09-05 00:34:49
tags: [iOS,网络,开发之术]
Categories: iOS
---

{% cq %}

我们需要检测用户的网络IP地址是否开启了代理（VPN），防止一些用户通过关闭GPS，让我们的IP定位错误，非法获取我们限制了地域的优惠，或者是传递虚假网络信息。

{% endcq %}



<!-- more -->


{% note primary %}
可以通过判断“**tap** **tun** **ipsec** **ppp**” 这些字段是否在“**CFNetworkCopySystemProxySettings**”里，来判断**vpn**是否开启。
{% endnote %}

完整方法在此：

```Objc
- (BOOL)isVPNOn
{
   BOOL flag = NO;
   NSString *version = [UIDevice currentDevice].systemVersion;
   // need two ways to judge this.
   if (version.doubleValue >= 9.0)
   {
       NSDictionary *dict = CFBridgingRelease(CFNetworkCopySystemProxySettings());
       NSArray *keys = [dict[@"__SCOPED__"] allKeys];
       for (NSString *key in keys) {
           if ([key rangeOfString:@"tap"].location != NSNotFound ||
               [key rangeOfString:@"tun"].location != NSNotFound ||
               [key rangeOfString:@"ipsec"].location != NSNotFound ||
               [key rangeOfString:@"ppp"].location != NSNotFound){
               flag = YES;
               break;
           }
       }
   }
   else
   {
       struct ifaddrs *interfaces = NULL;
       struct ifaddrs *temp_addr = NULL;
       int success = 0;
       
       // retrieve the current interfaces - returns 0 on success
       success = getifaddrs(&interfaces);
       if (success == 0)
       {
           // Loop through linked list of interfaces
           temp_addr = interfaces;
           while (temp_addr != NULL)
           {
               NSString *string = [NSString stringWithFormat:@"%s" , temp_addr->ifa_name];
               if ([string rangeOfString:@"tap"].location != NSNotFound ||
                   [string rangeOfString:@"tun"].location != NSNotFound ||
                   [string rangeOfString:@"ipsec"].location != NSNotFound ||
                   [string rangeOfString:@"ppp"].location != NSNotFound)
               {
                   flag = YES;
                   break;
               }
               temp_addr = temp_addr->ifa_next;
           }
       }
       
       // Free memory
       freeifaddrs(interfaces);
   }


   return flag;
}
```

这个是判断系统的vpn状态，还有一种是调起vpnExtension或着说NEVPNManager的时候的vpn状态可以通过实例化NEVPNManager，然后通过manager.connection.status来获取当前的vpn状态。


该方法有一个缺陷，就是如果用户是连接了已经开启代理的**Wi-Fi**网络，那我们就无法检测到啦！如果有更好的方法欢迎留言告知～