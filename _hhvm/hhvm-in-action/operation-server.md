## 1. 背景

HHVM 是一款高性能的PHP虚拟机，但是目前由于国内应用的人群少，运维经验少，所以针对此问题我们将百度的HHVM运维方式进行一些讲解。

我们在已经安装好HHVM，并且可以根据推荐配置运行HHVM基础上，那么我们就要真正上线HHVM了，那么如何保证HHVM的线上安全呢？
在运维安全上我们主要依赖于2点：

 1.	进程存活状态
 1.	HHVM监控

上线和迁移方式：

 1. jit 预热
 1. 旁路迁移方式

下面具体的介绍如上内容：

## 2.	服务安全

HHVM的web运行方式和PHP不同，是采用单进程多线程的模式，多线程模型有他的好处就是消耗资源少、节约内存、并发性高，但是也有他的劣势，那么就是线程方式容易出现线程安全问题、内存泄露、crash后整个进程退出；
虽然有如上弊端，但是我们可以通过一些方式进行规避，至少目前在百度的运维过程中，这些方案还是比较稳定的，如下我们通过几种方式进行避免和定位一些线上异常现象：

### 1. crash问题
hhvm 如果出现crash后，其实对于服务来说是很危险的，相对PHP的多进程模式crash一个slave进程无所谓，但是如果hhvm crash了，那么整个进程就退出了，那么我们如何避免这种问题呢？
我们用一些工具，或者自己写一些监控工具，监测HHVM的进程存活状态（如surpervise），当进程crash后即时拉起避免期间所带来的流量损失，但是crash必定会造成一定的流量丢失，而且HHVM还会有预热过程（JIT翻译时一般比直接解释执行还要慢不少），那么我们如何针对这些问题进行避免呢？

  1.	web server（如nginx、lighttpd等）代理hhvm，通过策略调度到备机
    
  通过upstream或者error_page等方式切换到备机上,初期的话备机是ZEND虚拟机，但是后期也打算备机也换成HHVM，由于当HHVM crash后，切换到zend后会出现cpu瞬间过高或者打满cpu的现象，所以为了避免这种情况，可以预先预热好一个HHVM作为备机，然后采用这种双buffer的机制进行运行，这种情况下就可以在HHVM crash的情况下也可以保证流量不丢失。
    
  1. 内存泄露问题和线程安全问题将在如下2篇详解

    
### 2. 内存泄露问题


  内存泄露对于多线程模式还是比较常见的，一般在上线前我们会针对内存泄露进行一系列的测试，但是如果线上出现了内存泄露我们如何处理呢？
  具体分析过程我们已经在内存泄露分析篇中进行介绍这里不进行详细阐述，我们主要针对内存泄露在线上出现时一些处理方式：
  
  1.	内存监控报警
  1.	设置MaxRss进行内存约束，超过内存后HHVM退出，通过supervise拉起
  1.	线下复现和分析内存泄露
  
### 3.线程安全问题
  
  线程安全问题在多线程模式下也是一种比较常见的现象，而且此种问题比较危险，线程安全问题一般会出现如下现象：
  
  1.	crash
  1.	死锁
  1.	信息不一致
  
  那么我们将介绍下如何针对上面3种情况在线上的解决方式：
  
####	1. Crash
  
  线程安全问题中，crash应该是最好解决的一种，因为crash后，我们可以直接看到栈，然后分析究竟是哪里出现了问题，一般线程安全crash都是针对全局的静态变量操作并且销毁时会遇到此种问题，如onig库默认是单进程模式，如果此类问题，我们只需要查看crash 栈即可分析。
  
####  2.	死锁
  
  死锁问题是线上遇到问题的一种棘手的问题，但是也还好分析，为何说棘手呢？因为出现死锁时，会造成大范围请求的延迟，无法接收新的请求，那么我们如何分析线上是否出现了死锁呢？
  
 未完今天添加……