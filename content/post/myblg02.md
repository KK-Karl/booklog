---
title: "redis限流方案汇总"
author: ""
type: ""
date: 2024-04-14T13:47:36+08:00
subtitle: ""
image: ""
tags: ["技术","中间件"]

draft: false

---

## 楔子

**"限流"这种事情即使在生活中也很常见，比如我们银行办理业务，银行不可能给去的所有人同时服务，因为柜台就那么几个。所以可能一次只给5个人办理业务，其他的人只能在后面排队；再比如打饭等等，也是一样的道理。因为能提供服务的数量有限，所以必须要通过限流的方式。**

**在程序的层面上也是一样的，如果我们的系统只能支持10万人同时在线购物，但是某一天突然来了100万个用户，那么后果显然是服务器直接瘫痪。因此只能让"限流"的功能来维护，先让一部分用户进行购物，其它的人进行排队，这样就等保证整个系统进行运转了。**

> **这里提一下微博，微博因为哪个明星出轨了，或者哪个明星恋爱官宣了，导致服务器挂掉不止一次了。这里也是因为人数骤增导致的，当然这只是表层的原因，关于背后详细的架构、以及为什么这种情况一出现微博就又可能会挂掉的原因，我可能会在其它的系列中说，这里先不谈，今天主要是谈限流。**
>
> **这里立一个flag，要是哪天所有人都以为还是单身的胡歌在微博上突然官宣："大家好，这是我的妻子xxx，我们昨天领证了，希望得到大家的祝福。"，然后再附带两人结婚照和结婚证，我跟你说微博必瘫痪，话就撂这了。**

## Redis如何实现限流功能

**关于限流所用的算法有两个：漏桶算法、令牌算法。**

### 漏桶算法

**漏桶算法的灵感源于漏斗，如下图所示：**

![img](https://img2020.cnblogs.com/blog/1229382/202007/1229382-20200716164904353-1015512408.png)

**首先，如果让你实现一个限流的算法你要怎么做呢？我们可以规定一个时间，比如60s，在60s之内只能处理100个请求，如果超过了100个，那么就将多余的请求丢弃掉。但是这样存在一个问题：如果在10s内请求就已经100个，因此剩余的50s只能把再来的请求给丢弃掉。但是这100个请求又花了10s中就全部处理完了，那么剩余的40s做什么？显然这样做就存在这资源浪费的情况。于是可能有人想到使用队列的方式，设置队列的容量为100，任务处理完了就出队，然后等待处理的入队，这样就保证了资源的利用率，恭喜你，漏桶算法就是这么做的。**

**说实话从漏斗本身上想，也能猜出使用的是队列，一边进一边出。无论漏斗上面的水流有多少，漏斗下面的水都是均匀流出的。如果上面的水流量大于下面流出的水流量的话，那么漏斗会慢慢变满；反之，漏斗永远不会被装满，并一直流出。**

**漏桶算法的实现步骤是：先声明一个队列保存请求，这个队列相当于漏斗，当队列容量满了之后，就放弃新来的请求。然后执行的话则是通过声明另一个线程定期从队列中获取一个或多个任务执行，这样就实现了漏桶算法。**

> **漏桶算法可以在编程语言这一层实现。**

### 令牌算法

**令牌算法指的是有一个程序以某种恒定的速度生成令牌，并存入令牌桶中，而每个请求需要先获取令牌才能执行，如果没有获取到令牌的话则放弃执行。如下图所示：**

![img](https://img2020.cnblogs.com/blog/1229382/202007/1229382-20200716164912827-1879589360.png)

**这种令牌算法，我们也可以使用Python的threading模块来实现。**

### 更好的限流方案

**在Redis4.0，已经为我们实现了限流功能，并提供了原子的限流指令，再加上Redis这个天生的分布式程序就可以完美地实现限流了。**

**实现限流需要使用Redis提供的Redis-Cell模块，该模块使用的便是漏桶算法，使用起来也很简单，通过cl.throttle即可，但是我们需要提前安装。**

**前往：`https://github.com/brandur/redis-cell/releases`下载对应系统的安装包，有源码编译和安装包安装两种方式，源码编译需要rust环境、比较复杂，所以推荐安装包安装。安装之后，直接解压即可，里面会有一个`libredis_cell.so`文件，执行redis-cli通过module load加载即可。**

```sh
> cl.throttle mylimit 15 30 60
1）（integer）0 # 0 表示获取成功，1 表示拒绝
2）（integer）15 # 漏斗容量
3）（integer）14 # 漏斗剩余容量
4）（integer）-1 # 被拒绝之后，多长时间之后再试（单位：秒）-1 表示无需重试
5）（integer）2 # 多久之后漏斗完全空出来
```

**其中 15 为漏斗的容量，30 / 60s 为漏斗的速率。**

**通过Redis-Cell，我们则可以实现分布式原子级别的限流，更详细的使用可以查看官网**