# DistributedFSTree
分布式文件系统

![](https://i.imgur.com/pn4POJY.png)

<pre>
FastDFS
       FastDFS是一个开源的轻量级分布式文件系统，它对文件进行管理，功能包括：
          1）文件存储
          2）文件同步
          3）文件访问（文件上传，文件下载）
       解决了大容量文件存储和负载均衡的问题。特别适合以文件为载体的在线服务，如相册网站，视
       频网站等。
  
       FastDFS为互联网量身定制，充分考虑了冗余备份，负载均衡，线性扩容等机制，并注重高
       可用，高性能等指标，使用FastDFS很容易搭建一套高性能的文件服务器集群提供文件上传，下载等服务。

       FastDFS特别适合中小文件（建议范围: 4KB ~ 500MB）的在线服务，如相册网站，视屏网站
       等等。

       FastDFS是一款开源的轻量级分布式文件系统纯C实现，支持Linux，FreeBSD等UNIX系统类
       Google FS,不是通用的文件系统，只能通过专有的API访问，目前提供了C，JAVA和PHP API
       为互联网量身定做，解决大容量文件存储问题，追求高性能和高扩展性FastDFS可以看做是基
       于文件的key-value pair存储系统，称作分布式文件存储服务更为合适。


       FastDFS服务端有3个角色：
           1）跟踪服务器（tracker server）
              主要做调度工作，起负载均衡的作用。在内存记录集群中所有存储组合存储服务器的
              状态信息，是客户端和数据服务器交互的枢纽，相比GFS中的master更为精简，不记
              文件索引信息，占用的内存量很少。 

              Tracker是FastDFS的协调者，负责管理所有的Storage Server和group，每个
              Storage在启动后会连接Tracker，告知自己所属的group等信息，并保持周期性
              的心跳，tracker根据storage的心跳信息，建立group -> storage list的
              映射表。

              Tracker需要管理的元信息很少，会全部存储在内存中；另外tracker上的元信息
              都是由storage汇报的信息生成的，本身不需要持久化任何数据，这样使得tracker
              非常容易扩展，直接增加tracker机器即可扩展tracker cluster来服务，cluster
              里每个tracker之间是完全对等的，所有的tracker都接收storage的心跳信息，生
              成元数据信息来提供读写服务。

           2）存储服务器（storage server）
              又称存储节点或数据服务器，文件和文件属性都保存到存储服务器上，
              Storage Server直接利用OS的文件系统调用管理文件。

              Storage Server以组（卷 group或volume）为单位组织，一个group内包含多台
              Storage机器，数据互为备份，存储空间以group内容量最小的storage为准，所以
              建议group内的多个storage尽量配置相同，以免造成存储空间的浪费。

              以group为党委组织存储能方便的进行应用隔离，负载均衡，副本数定制（group
              内storage server数量即为该group的副本数），比如将不同应用数据存到不同
              的group就能隔离应用数据，同时还可根据应用的访问特性来将应用分配到不同的
              group来做负载均衡;缺点是group的容量受单机存储容量的限制，同时group内
              有机器坏掉时，数据恢复只能依赖group内的其他机器，使得恢复时间会很长。

              group内每个storage的存储依赖本地文件系统，storage可配置多个数据存储目录
              ，比如有10块磁盘，则都可配置为storage的数据存储目录。

              storage接收到写文件请求时，会根据配置好的规则，选择其中一个存储目录来存储
              文件。为了避免单个目录下的文件数太多，在storage第一次启动时，会在每个数据
              存储目录里创建2级子目录，每级256个，总共65536个文件，新写的文件会以hash的
              方式被路由到其中某个子目录，然后将文件数据直接作为一个本地文件存储到该目录中。

           3）客户端（client）
              客户单是作为业务请求的发起方，通过专有接口，使用TCP/IP协议域跟踪服务器或存
              储节点进行数据交互。FastDFS向使用者提供基本文件访问接口，比如:
                 upload, download,append,delete等
              以客户端库的方式提供给用户使用。

      group:组，也可称为卷.同组内服务器上的文件是完全相同的，同一组内的storage server
            之间是对等的，文件上传，删除等操作可以再任意一台storage server上进行。
</pre>
