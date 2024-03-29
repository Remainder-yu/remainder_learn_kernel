
# 反向路径过滤
rpfilter是Linux内核中的一个网络过滤机制。它代表"Reverse Path Filter"，用于检查入站数据包的源IP地址是否通过正确的网络路径到达，以防止使用欺骗性IP地址的攻击。
rpfilter的作用是在路由决策之前进行检查，如果数据包的源IP地址不是通过正确的路径到达，rpfilter会将该数据包视为异常，并根据配置的策略进行处理。默认情况下，rpfilter会丢弃该数据包，但也可以根据需要进行其他处理，如记录日志或发出警告。
rpfilter的配置文件位于/proc/sys/net/ipv4/conf/*/rp_filter。以下是其中一些常见的配置选项：


0：关闭rpfilter，数据包不会进行检查。

1：开启rpfilter，仅检查数据包的入站接口是否相符，不会检查数据包的出站接口。

2：开启严格模式的rpfilter，检查数据包的入站接口和出站接口是否相符。
具体的配置可能因Linux发行版和内核版本的不同而有所不同。
总的来说，rpfilter是Linux内核中用于检查网络数据包的一个重要机制，它可以帮助防止欺骗性IP地址的攻击，并提高网络的安全性。

