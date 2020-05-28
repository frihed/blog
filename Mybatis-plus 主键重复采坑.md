# Mybatis-plus 主键重复采坑

## 背景：

Mybatis Plus的主键策略有多种，默认好像是 ID_WORKER，当插入对象ID 为空，才自动填充

在 k8s容器环境下，项目出现偶发性的自增ID冲突

检查mybatis plus源码，关键类：com.baomidou.mybatisplus.core.toolkit.Sequence ，采用的是Twitter的雪花算法(Snowflake)，从这里修改而来 [分布式高效有序ID生产黑科技(sequence)](https://gitee.com/yu120/sequence)。 默认使用无参构造函数

生成ID的结构为：

```java 
        // 时间戳部分 | 数据中心部分 | 机器标识部分 | 序列号部分
        return ((timestamp - twepoch) << timestampLeftShift)
            | (datacenterId << datacenterIdShift)
            | (workerId << workerIdShift)
            | sequence;
```

符号位1位，时间戳 41位，数据中心 5位，机器表示5位，序列号12位

检查数据标识 id 部分代码

```java
/**
     * 数据标识id部分
     */
    protected static long getDatacenterId(long maxDatacenterId) {
        long id = 0L;
        try {
            InetAddress ip = InetAddress.getLocalHost();
            NetworkInterface network = NetworkInterface.getByInetAddress(ip);
            if (network == null) {
                id = 1L;
            } else {
                byte[] mac = network.getHardwareAddress();
                if (null != mac) {
                    id = ((0x000000FF & (long) mac[mac.length - 1]) | (0x0000FF00 & (((long) mac[mac.length - 2]) << 8))) >> 6;
                    id = id % (maxDatacenterId + 1);
                }
            }
        } catch (Exception e) {
            logger.warn(" getDatacenterId: " + e.getMessage());
        }
        return id;
    }
```

这段代码 id 赋值部分有bug、而且有些代码怪怪的，比如为什么要先右移6位，之后取余？

因为看不到github上项目16年8月之前的提交历史，大胆推测：

1. 原算法 [分布式高效有序ID生产黑科技(sequence)](https://gitee.com/yu120/sequence) 只有三部分，中间10位机器码，可以支持1023台机器，这里最早的实现只有 datacenterId 没有 workerId ，所以从mac地址中右移六位取了10位（注意这里有bug）： 
    > id = ((0x000000FF & (long) mac[mac.length - 1]) | (0x0000FF00 & (((long) mac[mac.length - 2]) << 8))) >> 6
2. 因为有bug ，自增Id 容易冲突，所以后续迭代将这里的 datacenterId 取余保留了5位，增加了workerId 引入PID ，取余保留5位，来避免冲突
3. 之后依然发现有冲突，所以引入了有参构造器，将这个问题抛给用户


## bug 分析

在k8s 容器环境下，因为以下因素，导致最终生成的ID冲突的可能性还是比较大的

1. 因为环境一致 pid 一样的可能性很大
2. datacenterId的实现有bug
3. datacenterId 和 workerId 有取余操作


首先， Docker 容器的mac地址是根据 Ip生成的 [generateMacAddr](https://github.com/docker/docker/blob/15b6b7be010546f30d7eabd000167d428efc0b13/daemon/networkdriver/bridge/driver.go#L335-L358)， 前两位固定，后四位是ip地址：

```go
// Generate a IEEE802 compliant MAC address from the given IP address.
//
// The generator is guaranteed to be consistent: the same IP will always yield the same
// MAC address. This is to avoid ARP cache issues.
func generateMacAddr(ip net.IP) net.HardwareAddr {
	hw := make(net.HardwareAddr, 6)

	// The first byte of the MAC address has to comply with these rules:
	// 1. Unicast: Set the least-significant bit to 0.
	// 2. Address is locally administered: Set the second-least-significant bit (U/L) to 1.
	// 3. As "small" as possible: The veth address has to be "smaller" than the bridge address.
	hw[0] = 0x02

	// The first 24 bits of the MAC represent the Organizationally Unique Identifier (OUI).
	// Since this address is locally administered, we can do whatever we want as long as
	// it doesn't conflict with other addresses.
	hw[1] = 0x42

	// Insert the IP address into the last 32 bits of the MAC address.
	// This is a simple way to guarantee the address will be consistent and unique.
	copy(hw[2:], ip.To4())

	return hw
}

```

在局域网中，主要变动在后两位，而相近的 ip主要变动则在最后一位，比如 ：

- ip 172.30.208.3 ，生成的mac 地址位：02:42:ac:1e:d0:03
- ip 172.30.208.5 ，生成的mac 地址位：02:42:ac:1e:d0:05


源代码中：

```java
id = ((0x000000FF & (long) mac[mac.length - 1]) | (0x0000FF00 & (((long) mac[mac.length - 2]) << 8))) >> 6
```

取最后一位 和 左移8位的倒数第二位 取或，然后右移6位，此时最后一位（8bit）只有前2个bit对结果有影响，造成的问题是，若ip 相差较小，那么生成的 datacenterId 是一样的。比如ip范围： 172.20.208.1 - 172.20.208.63 ，生成的结果都是相同的，这是不应该的。


## bug 修复

若 datacenterId 不采用移位运算然后取余，而是直接取mac地址后10位，用这个值作为原算法 [分布式高效有序ID生产黑科技(sequence)](https://github.com/docker/docker/blob/15b6b7be010546f30d7eabd000167d428efc0b13/daemon/networkdriver/bridge/driver.go#L335-L358) 中的 10 位机器码：

```java
(0x000000FF & (long) mac[mac.length - 1]) | (0x00000300 & (((long) mac[mac.length - 2]) << 8)) ;
```

就可以保证在**22位掩码的子网中**，所有节点不冲突（注意：C类网络，主机标识的长度为8位，掩码24位；B类网络，主机地址16位，掩码16位）。


例： 172.20.1.1/22 ，ip 范围： 172.20.1.1- 172.20.3.254 共1022个节点。


## 临时的方案

子网掩码不小于22的局域网，直接取mac地址后10位即可，将datacenterId 的实现拷贝出来，修改关键代码，取后10位，分成两部分作为 workerId 和 datacenterId 传入：

```java
  @PostConstruct
  public void initIdWorker() {
    long id = 0L;
    try {
      InetAddress ip = InetAddress.getLocalHost();
      NetworkInterface network = NetworkInterface.getByInetAddress(ip);
      if (network == null) {
        id = 1L;
      } else {
        byte[] mac = network.getHardwareAddress();
        if (null != mac) {
          id = ((0x000000FF & (long) mac[mac.length - 1]) | (0x00000300 & (
              ((long) mac[mac.length - 2]) << 8)));
        }
      }
    } catch (Exception e) {
      log.warn(" getDatacenterId: " + e.getMessage());
    }
    final long workerId = id & 0x1f;
    final long datacenterId = (id >> 5) & 0x1f;
    log.info("workerId: {}, datacenterId: {}", workerId, datacenterId);
    IdWorker.initSequence(workerId, datacenterId);
  }

```

子网掩码大于22的局域网，比如在 B类网络中，你的应用可能分配任何ip的话，有 2^16-2中可能，就不是10位机器码能够自然标识的了，不依赖任何配置信息和数据记录的情况下，总有可能生成一样的机器码。

其实作为通用实现来说，机器码10位有点少，时间戳41位太长（数据保存69年，开啥玩笑呢），序列号12位太长（1毫秒支持产生4095个自增序列id，这并发针对 twitter的峰值设计，对于大多数应用来说太浪费了）。
