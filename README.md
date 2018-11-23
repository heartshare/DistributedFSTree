# DistributedFSTree
分布式文件系统

![](https://i.imgur.com/pn4POJY.png)

文件上传机制

![](https://i.imgur.com/EkrYOaE.png)

文件下载机制

![](https://i.imgur.com/dYNugQm.png)

同步信息上报图

![](https://i.imgur.com/T5nEkIs.png)

快速文件定位查询

![](https://i.imgur.com/UQ7K8Xu.png)

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

      Tracker相当于FastDFS的大脑，不论是上传还是下载都是通过tracker来分配资源；客户端
      一般可以通过使用nginx等静态服务器来调用或者做一部分的缓存；存储服务器内部分为卷
      或者叫组，卷与卷之间是平行的，可以根据资源的使用情况随时增加，卷内服务器文件相互
      同步备份，以达到容灾的目的。

      上传机制：
          首先客户端请求tracker服务获得存储服务器的IP地址和端口，然后客户端根据返回的IP
      地址和端口号请求上传文件，存储服务器接收到请求后生成文件，并且将文件内容写入磁盘并
      返回给客户端file_id, 路径信息，文件名等信息，客户端保存相关信息上传完毕。

          1、选择tracker server
            当集群中不止一个tracker server时，由于tracker之间是完全对等的关系，客
            户端在upload文件时可以任意选择一个trakcer。 选择存储的group 当tracker接
            收到upload file的请求时，会为该文件分配一个可以存储该文件的group，支持如
            下选择group的规则：
                1、Round robin，所有的group间轮询
                2、Specified group，指定某一个确定的group
                3、Load balance，剩余存储空间多多group优先

          2、选择storage server
            当选定group后，tracker会在group内选择一个storage server给客户端，支持如
            下选择storage的规则：
                1、Round robin，在group内的所有storage间轮询
                2、First server ordered by ip，按ip排序
                3、First server ordered by priority，按优先级排序（优先级在storage
                  上配置）

          3、选择storage path
            当分配好storage server后，客户端将向storage发送写文件请求，storage将
            会为文件分配一个数据存储目录，支持如下规则：
                1、Round robin，多个存储目录间轮询
                2、剩余存储空间最多的优先
                3、生成Fileid
            选定存储目录之后，storage会为文件生一个Fileid，由storage server ip、
            文件创建时间、文件大小、文件crc32和一个随机数拼接而成，然后将这个二进制串进
            行base64编码，转换为可打印的字符串。 选择两级目录 当选定存储目录之
            后，storage会为文件分配一个fileid，每个存储目录下有两级256*256的子目
            录，storage会按文件fileid进行两次hash（猜测），路由到其中一个子目录，
            然后将文件以fileid为文件名存储到该子目录下。

         5、生成文件名
            当文件存储到某个子目录后，即认为该文件存储成功，接下来会为该文件生成一个文
            件名，文件名由group、存储目录、两级子目录、fileid、文件后缀名（由客户端指
            定，主要用于区分文件类型）拼接而成。

     下载机制：
         跟upload file一样，在download file时客户端可以选择任意
         tracker server。tracker发送download请求给某个tracker，必须带上文件名
         信息，tracke从文件名中解析出文件的group、大小、创建时间等信息，然后为该请
         求选择一个storage用来服务读请求。由于group内的文件同步时在后台异步进行的，
         所以有可能出现在读到时候，文件还没有同步到某些storage server上，为了
         尽量避免访问到这样的storage，tracker按照如下规则选择group内可读的storage。

           1、该文件上传到的源头storage - 源头storage只要存活着，肯定包含这个文件，
             源头的地址被编码在文件名中。

           2、文件创建时间戳==storage被同步到的时间戳 且(当前时间-文件创建时间戳) > 0; 文件同步最大时间（如5分钟) - 文件创建后，认为经过最大同步时间后，肯定已经同
           步到其他storage了。

           3、文件创建时间戳 &lt; storage被同步到的时间戳。 - 同步时间戳之前的文件确定
           已经同步了

           4、(当前时间-文件创建时间戳) &gt; 同步延迟阀值（如一天）。 - 经过同步延迟阈
           值时间，认为文件肯定已经同步了

     同步时间管理：
         当一个文件上传成功后，客户端马上发起对该文件下载请求（或者删除请求）时，tracker
     是如何选定一个适用的存储服务器呢？其实每个存储服务器都需要定时将自身的信息上报给
     tracker，这些信息就包括本地同步时间（即同步到最新文件的时间戳）。而tracker根据各个
     存储服务器的上报情况，就能够知道刚刚上传的文件，在该存储组中是否完成了同步。同步信息
     上报如图：

         写文件时，客户端将文件写至group内一个storage server即认为写文件成功，storage
     server写文件后，会由后台线程将文件同步至同group内其他的storage server.

         每个storage写文件后，同时会写一份binlog，binlog里不包含文件数据，只包含文件名
     等元信息， 这份binlog用于后台同步，storage会记录向group内其他storage同步的进度，
     以便重启后能接上次的进度继续同步；进度以时间戳的方式进行记录，所以最好能保证集群内
     所有server的时钟保持同步。

        storage的同步进度会作为元数据的一部分汇报到tracker上，tracker在选择读storage
     的时候会以同步进度作为参考。比如一个group内有A,B,C三个storage server,A向C同步进度
     为T1（T1以前写的文件都已经同步到B上），B向C同步到时间戳为T2，tracker接收到这些同步
     进度信息时，就会进行整理，将最小的那个作为C的同步时间错，根据上述规则，tracker会为
     A,B生成一个同步时间戳。

     精巧的文件ID-FID
         说到下载文件就不得不提文件索引了（又称FID）的精巧设计了。文件索引结构如下图，是
     客户端上传文件后存储服务器返回给客户端，用于以后访问文件的索引信息。文件索引信息包括
        1）组名  2） 虚拟磁盘路径  3）数据两级目录  5）文件名
     其中文件名与文件上传时不同，是由存储服务器根据特定信息生成，文件名包含：
        源存储服务器IP地址，文件创建时间戳，文件大小，随机数，文件拓展名等信息

     快速文件定位：
        知道FastDFS FID的组成后，我们来看看FastDFS是如何通过这个精巧的FID定位到需要
     访问的文件的。
        1：通过组名tracker能够很快的定位到客户端需要访问的存储服务器组。并将选择合适的
           存储服务器提供给客户端访问。
        2：存储服务器根据“文件存储虚拟磁盘路径”和“数据文件两级目录”可以很快定位到文件
           所在目录，并根据文件名找到客户端需要访问的文件。
</pre>
