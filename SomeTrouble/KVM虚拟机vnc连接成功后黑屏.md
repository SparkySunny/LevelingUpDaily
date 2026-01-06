## KVM无法创建虚拟机 
我在创建KVM虚拟机时出现了一个问题，不管是VNC还是什么console都无法操控创建的虚拟机，创建centos时候

>BUG: soft lockup - CPU#3 stuck for 23s!

我在排查错误又换了三个镜像后，用centos试出来的报错，这个报错去论坛里面搜了一下，找到了一篇**关于kvm安装Linux时的CPU soft lockup报错解决方案**的文章，于是我突然想是不是我的KVM和我的宿主机u
