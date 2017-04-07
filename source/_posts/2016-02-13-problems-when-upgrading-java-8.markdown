---
layout: post
title: "升级至Java 8遇到的问题系列之tls1.2"
date: 2016-02-13 18:24:55 +0800
comments: false
categories: 
---
##背景
公司中所有项目均使用java 6，现要升级至java 8。项目G中需要调用一个外部web service(以下简称**S**)。该web service的协议是https。项目G使用apache-cxf作为jax-ws client.部署在jetty 6上。
###问题表象
项目G升级至java 8后，调用S，第一次能成功得到响应以及正确的结果。第二次及其以后的调用得到的结果都是read timeout。
###初步分析
此次除开升级java 8，其他代码均未改变，所以很容易想到是升级java 8后导致的。所以我就google了一下ssl在java 8中的变化。

[JDK 8 will use TLS 1.2 as default](https://blogs.oracle.com/java-platform-group/entry/java_8_will_use_tls)这篇oracle官方的博客提到java 8会默认使用tls1.2，而java 6是默认使用tls1.0，在jvm启动参数中加入-Dhttps.protocols=TLSv1可以将tis版本指定运行在1.0下，加入这个参数后read timeout消失了。这个证明确实是由于tis版本的升级导致的此次问题的发生。但是这个照理来说应该不会是个问题，继续深入分析。
###深入分析
遇到这种协议上的问题，我一般会通过抓包来分析一下根本原因。所以我又通过Wireshark抓包来对比了一下项目运行在java 6和java 8下的区别。

![1.8](../images/1.8_tls_capture.png)
从上图红框部分我们可以看出，在第三次client hello包发出后，得到的反馈是一个ack。

我们再看下java 6时的抓包截图，可以看出发出第三次client hello后得到的反馈是一个server hello.

![1.6](../images/1.6_tls_capture.png)

从上面的截图对比可以看出，运行在java 8的时候，第三次client hello，S端不认识这条信息。通过分析运行在java 8下的第一，二次client hello包内容和第三次client hello包内容的区别时，第三次会包含一个session id，但是java 6下也会有这个session id。

我又重新梳理了一遍之前分析的资料，又看了[JDK 8 will use TLS 1.2 as default](https://blogs.oracle.com/java-platform-group/entry/java_8_will_use_tls)这篇博客一遍，发现里面提到了一个可以测试目标网站对于tls1.2支持的情况的[网站](https://www.ssllabs.com/ssltest/analyze.html)。对S的测试结果如下图：

![tls_analyze](../images/java_8_tls_problem/tls_analyze_for_gs.png)

从上图红框处，我们可以看出这个不支持[Long handshake](https://github.com/ssllabs/research/wiki/Long-Handshake-Intolerance)，即当包当中hello message超过256字节时，会不认识这个消息包。。。，回头又分析了一下运行在java 8下的包，52 cipher suites 加上session id就会是的这个message的大小变为271字节，从而让server端解析这个包失败。

###解决方案
通过在cxf.xml中配置一个cipherSuiteFilter去掉不必要cipher suite从而使得message的长度变短就根本解决了这个问题。