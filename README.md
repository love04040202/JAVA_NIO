# JAVA_NIO

##netty 源码分析  
https://segmentfault.com/a/1190000007282789

##Java NIO系列教程（六） Selector  
http://ifeve.com/selectors/#SelectionKey  
form http://blog.csdn.net/zavakid/article/details/6732082


Java NIO的实现中，有不少细节点非常有学习意义的，就好比下面的三个点：
1） Selector的 wakeup原理是什么？是如何实现的？
2） Channel的close会做哪些事？
3） 会什么代码中经常出现begin()和end()这一对儿？

本文虽然针对这几个点做了点分析，不能算是非常深刻，要想达到通透的地步，看来还得经过实战的洗练。

1、 wakeup()
准确来说，应该是Selector的wakeup()，即Selector的唤醒，为什么要有这个唤醒操作呢？那还得从Selector的选择方式来说明，前文已经总结过Selector的选择方式有三种：select()、select(timeout)、selectNow()。
selectNow的选择过程是非阻塞的，与wakeup没有太大关系。
select(timeout)和select()的选择过程是阻塞的，其他线程如果想终止这个过程，就可以调用wakeup来唤醒。

wakeup的原理
既然Selector阻塞式选择因为找到感兴趣事件ready才会返回(排除超时、中断)，就给它构造一个感兴趣事件ready的场景即可。下图可以比较形象的形容wakeup原理：

Selector管辖的FD(文件描述符，Linux即为fd，对应一个文件，windows下对应一个句柄；每个可选择Channel在创建的时候，就生成了与其对应的FD，Channel与FD的联系见另一篇)中包含某一个FD A， A对数据可读事件感兴趣，当往图中漏斗端放入(写入)数据，数据会流进A，于是A有感兴趣事件ready，最终，select得到结果而返回。

#Java NIO——Selector机制解析三（源码分析）  
http://goon.iteye.com/blog/1775421  
    可见创建的ServerSocketChannelImpl也有WindowsSelectorImpl的引用。
    Java代码  收藏代码
    ServerSocketChannelImpl(SelectorProvider sp) throws IOException {  
            super(sp);  
            this.fd =  Net.serverSocket(true);  //打开一个socket，返回FD  
            this.fdVal = IOUtil.fdVal(fd);  
            this.state = ST_INUSE;  
    }  

    然后通过serverChannel1.register(selector, SelectionKey.OP_ACCEPT);把selector和channel绑定在一起，也就是把new ServerSocketChannel时创建的FD与selector绑定在了一起。
    到此，server端已启动完成了，主要创建了以下对象：
    WindowsSelectorProvider：单例
    WindowsSelectorImpl中包含：
        pollWrapper：保存selector上注册的FD，包括pipe的write端FD和ServerSocketChannel所用的FD
        wakeupPipe：通道（其实就是两个FD，一个read，一个write）
