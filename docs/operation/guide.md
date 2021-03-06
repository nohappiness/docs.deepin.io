<!--Meta
category:系统运维
title:Server Operation
DO NOT Delete Meta Above -->

# 服务器开发与运维指导手册

## 概述

--------------------

本文档的目的不在于提供一整套完整的流程供运维人员与开发人员遵循，而是提供指导意见，以避免在开发与运维过程中出现太大的错误。

本文档针对的人群包括服务器开发人员以及服务器运维人员，以及可能需要通过服务器对外发布内容的人员。

## 运维安全

--------------------

为了保证服务的稳定安全，需要保证生产服务器的软硬件稳定安全，因此在操作上要：

### 服务器配置

* 保证生产服务器只有服务器运维人员可以登录，开发人员不可以登录。如果开发人员需要调试，可以通过日志与监控系统进行调试，运维人员可以将日志与其它的运行信息下载回来传递给开发人员，或者通过限制IP的Web界面让开发人员读取实时信息(例如用[Logstash](https://www.elastic.co/products/logstash)与[Elasticsearch](https://www.elastic.co/downloads/elasticsearch)搭建一个实时日志搜索网站，但是一定要限制IP或者通过证书限制，不然所有人都会看到了)
* 禁止root用户远程登录，通过各PAM模块设置密码强度，设置密码失败后的禁止登录时间，设置每个用户的最大资源阈值(包括CPU占用、打开文件数、磁盘空间限额等)，此外可以定期使用[john](http://www.openwall.com/john/)尝试破解当前服务器各用户的密码，如果在主流机器上一天之内通过无密码文件方式破解不了，则可以认为密码初步安全
* 尽量只开设80(HTTP)与443(HTTPS)这两个公知端口，并将SSH端口设为一个较大的数，避免轻易被扫描到，并通过nmap与curl验证通过缺省扫描无法获得服务器的详尽信息，包括服务器内核版本、SSH服务器与Web服务器的型号与版本、Web服务器开发语言(例如PHP)与第三方库的版本，并通过iptables进行二次限制，保证只有这三个端口开放
* 服务器的公用账号不允许登录(例如Web服务器的用户)，每个用户都只能使用自己的账号登录，通过sudo运行root命令，将SSH日志与sudo日志通过网络实时备份(传送)到备份服务器，若可能，则只允许公钥证书登录，禁止密码登录
* 通过脚本自动检查系统安全补丁，并仔细查看安全补丁，看看是否需要升级，补丁不仅是操作系统的补丁，也包括其上运行的软件的补丁，例如nginx、PHP等
* 定时检查并删除不用的用户、cronjob以及daemon

下面是上述的几个命令：

* 使用john破解密码：`unshadow /etc/passwd /etc/shadow > mypasswd && john mypasswd`
* 使用nmap扫描系统信息与端口：`nmap -A -v <your-host-ip>`
* 使用curl获取网站首页HTTP头部信息检查敏感信息：`curl -I -v http://<you-website>/`

### 备份恢复

需要进行广泛的备份，并制定恢复策略：

* 配置文件的备份，包括hosts文件、PAM配置文件、cron列表、Web服务器的配置文件、iptables配置规则、数据库配置文件等，简而言之，即一切由运维人员手工编辑修改的信息(自动生成的信息不需要备份)，都需要进行备份，这种备份推荐使用git或者hg来做，以便进行集中管理与回滚。如果服务器数量较大，可以考虑使用salt、puppet、chef或者ansible之类的工具进行批量配置与管理
* Web服务器日志以及其他与业务系统紧密相关的日志文件需要进行压缩后的长期备份，一般备份时间至少是2年，当然更长时间更好
* 数据库数据需要与开发人员协商决定备份策略与步骤，常见的策略是一周进行一次全备份，每天进行一次增量备份，可以保留最近一个月的所有备份
* 备份数据与生产数据不应在同一台服务器上，以避免单点故障，此外，也可以保证一旦生产服务器出现故障能进行快速转移，将生产环境在另一台服务器上快速建立起来。当然，如果使用Docker或者虚拟机技术，生产环境的转移会更快
* 需要制定恢复计划，并定期演练(例如一周一次)，以保证当生产服务器宕机时可以在规定时间内在另一台服务器上恢复起来(包括域名指向的修改以及CDN的通知)
* 如果条件许可，可以制定降级恢复计划，即保证网站有最小功能可以被用户访问，但是高级功能无法使用，例如用户可以读取旧帖，但是不能发帖，此时还需要考虑发布通告的处理(即通知用户网站暂时降级了，需要等待大概多长时间能够恢复等，这需要开发者留下预留的接口)
* 如果条件允许，也可以一直打开备份服务器上的服务，每次部署生产服务器时同时也部署备份服务器，这样在紧急恢复的时候一般只需要恢复数据库数据即可
* 备份服务器也可以用来做正式上线之前的测试服务器用
* 由于ICP备的原因，备份服务器需要和主服务器放在同一个托管商那里，但是尽量不放在一个机房里以避免机房级别的故障(断网、断电、DDoS、封禁...)
* 如果是阿里云等云公司的服务器，可以考察其备份恢复服务，看是否满足要求

### 监控报警

另外，要使用各种实时监控与报警手段：

* 外部第三方监控可以使用[监控宝](http://www.jiankongbao.com/)等产品，或者如果是阿里云的服务器，可以使用阿里云的监控报警服务
* 内部监控报警可以使用nagios、cacti、ganglia、zabbix等开源产品，较新的例如小米的[Open-Falcon](http://open-falcon.com/)均可使用
* 设立的报警阈值一定要有意义，报警应该是真正需要人手工处理的问题，如果一个报警阈值不需要处理，那就不要设置报警，不然会造成狼来了的现象，报警也就失去意义了

## 故障处理

--------------------

* 在发生故障时必须首先解决故障，而 **不是** 诊断问题，即首先尝试排除一切障碍让服务重新正常运行起来，事后再了解故障原因和解决深层次问题
* 若故障无法短期内解决，则需要启用备份服务器，将数据与代码转移到备份服务器，需要注意的问题是备份服务器的域名、CDN设置以及服务器资源(CPU、内存、存储、带宽)要能满足基本的运行条件
* 若故障无法短期内解决，也可以尝试回滚到最近的有效版本，包括配置、代码与数据库数据均需要回滚
* 事后必须要 **竭尽全力** 找到事故的原因，重现事故，更新代码或者流程，确保在同样的条件下不会再次出现此问题
* 大流量网站宕机之后重启需要使用灰度重启，即分时段分IP放开流量访问，以保证瞬间的峰值流量不至于冲垮服务器，导致循环宕机

## 性能

--------------------

* gzip，gzip，gzip，重要的事情要说三遍
* 设计好各种缓存与超时，主要是页面的客户端缓存(Expires、Etag、Last-Modified等几个HTTP头部需要好好掌握)、动态页面使用的memcache数据缓存，另外为了更新缓存，可以在资源后面加上版本信息，即加上?v=123之类的，其中123最好是版本号
* 手工检查高频访问页面(比如首页)，确保删除一切不需要的CSS/JS等外部链接
* 将不同类的内容分域名存放，至少需要将img/css/js与网页的域名分开，这样可以：
	* 使得浏览器能同时连接多个域名下载内容
	* 避免将cookie传到根本不需要的HTTP请求里去
	* 方便超时设置与内容管理，例如单独将图像、CSS和js放到CDN上，动态页面仍然放在主站上
	* 但是也不能设置太多域名，因为域名查找也花费时间
* 尽量将CSS放在网页头部，js放在网页尾部
* 对小的图片，可以使用内嵌图片，就是src=data:image/png;base64,...之类的
* 小图片太多的时候可以使用CSS Sprite
* 大图片尽量不要放在第一屏，而且可以使用延迟加载的技术载入
* 若可能，尽量将总是一起出现的CSS打包成一个文件，一起出现的JS打包成一个文件
* 使用gcc(Google Closure Compiler)对javascript进行编译精简与混淆
* 在每个域名的根目录都放一个favicon.ico并设置适当的超时，避免浏览器不停地请求这个文件

## 上线

--------------------

* 上线尽量安排在网站流量最少的时候(一般是4点左右)，以避免文件更新不一致导致用户看到错位的页面，这也要求我们上线要自动化
* 上线不能安排在周一，避免开发人员忘记之前的工作内容，也不能安排在周五，以免上线出现问题后没有人可以及时解决，因此上线时间只能安排在周三左右
* 上线必须让产品经理、运维人员和开发人员 **事先** 都知晓而且同意，而且必须要有产品经理与开发人员撰写的变更项说明，以及运维人员参与编写的部署说明
* 上线之前做好流量预估，这样好预先分配带宽，流量可以从历史访问记录来计算(旧网站)，或者上线之前先分配一个猜测带宽，再时刻准备好随时申请新带宽，同样的道理也适用于其他的动态资源(CPU与内存)预估
* 如果上线的是一个新域名，用来替换旧域名网站的，需要将原域名的URL以一种方式映射到新域名的网站来(比如使用nginx的rewrite功能)，当然，这种映射应该进行时常的检查，定期去掉已经没有人访问的链接，还有尽量手功能修改外部链接的就修改外部链接。原链接的来源可以来自于对原网站最近一段时间的日志的分析，获取外部网站对原网站最频繁的依赖。还需要向市场宣传的同事了解外部的广告宣传链接、友情链接或者内部的频道链接，以避免出现空链或者搜索引擎降权
* 每次上线都要保证外部统计都与原来的统计代码一致

## 开发与测试

--------------------

* 规划好git里面的程序路径，使得域名划分更方便，一般来说最好按照网站的主域名(如www、blog等)划分目录，然后静态文件(例如img/css/js)的域名按照主域名进行合并
* 为了减少由于开发环境与测试环境的差异导致的配置信息更改，所有的地址都尽量使用域名，而不是IP来访问，包括数据库的链接地址，然后通过本地的hosts来配置具体的IP
* 由于可能使用多域名开发网站，因此开发人员需在本机要有多套hosts文件，在开发与测试的时候进行替换以便开发与测试，当然，如果使用虚拟机和Docker会更方便

## 统计

--------------------

* 在客户端一定不能干扰正常功能，需要异步统计，不然在用户网络情况不好的时候会导致用户觉得正常功能很慢。同样，服务器端不能通过一个动态程序来处理统计请求，而是应该让客户端访问一个大小为0的静态文件，例如x.png来记录统计信息，再通过crobjob的方式定期读取新增的日志信息进行统计。也就是说客户端和服务器端的统计都是异步的，**绝不是**同步的
* 向服务器发的统计请求如果是通过浏览器发的，要注意在Web服务器里设置对应的统计URL的超时为0，这样以避免客户端浏览器的缓存导致服务器无法收到请求
* 为了方便Web服务器的日志采集，需要使用HTTP GET请求
* 如果用户的具体信息不重要，可以使用salted hash，比如salted sha2，这样可以保证用户隐私不会被暴露。但是我觉得我们大部分时候不需要这样做。另一个更好的方法是我们使用RSA加密用户信息后再通过base64之类的方法把数据传回服务器端，这样即使有人破解了客户端的代码，也无法得到明文的用户信息(还有一个附带的好处就是别人想伪造统计数据也更麻烦)。当然如果使用了hash或者加密的手段，每个统计日志除了原始文件，还需要保留一个解密以后的明文统计日志并加以备份，不然以后出现紧急任务手动查询的时候会不方便，也不利于其它工具的处理
* 为了保证统计信息尽量简短，不要浪费带宽和时间，统计服务器最好使用一个单独的域名，比如st.deepin.io等。这样可以保证统计信息不会带不需要的cookie，也可以加快统计信息的分析。同样的理由，我们也应该缩短请求的URL，比如GET请求里的每个字段的名字需要短一点，例如action可以简写成a之类的。还有就是冗余信息不需要传递，比如用户的外部IP地址，但是HTTP请求中一些重要的头部信息需要设置一下Web服务器的日志格式记录下来，比如x-forwarded-for这样的头部
* 原始统计数据不能被删除，应该tgz一下按日期保存好，这样一旦以后需要抽取更多信息或者原有的统计代码有误或者数据库损坏，都有最原始的东西可以重新来做，在需要临时查询数据的时候，一般来回使用`awk`、`sort`、`uniq`就可以了
*  数据库不是统计日志的备份，应该是后台统计系统上需要什么功能就保存什么数据，这样系统性能会有成百上千倍的提升。例如界面上如果只需要统计每天的IP和PV数，那每天只需要在数据库增加一条记录即可。**绝对不能**把所有数据保存进数据库，然后来一个count查询来做
* 对统计数据的保存，可以不使用MySQL之类的通用数据库，而是使用时间序列数据库，比如通过rrdtool来做，或者OpenTSDB等，这种数据库是专门为统计数据而设计的
* 对于统计系统最关键或最常用的数据，一般都是事先生成，而不是临时到数据库里面去查询的，比如统计系统的首页数据等
* 如果没有实时性要求，从日志到统计系统数据的归总一般都是安排在流量最低的时候进行，以避免无谓的性能消耗
* 外部统计一般都是放在页面底部，而且要通过一个不可见的DIV将其包含起来

## 自动化

--------------------

上述的所有工作，能自动化的时候就自动化，包括：

* 自动上线，每天设定上线检查时间，检查版本的差异，将需要上线的数据打包通过scp(证书认证)传输到待上线服务器展开，执行差异化操作(不仅仅是解包复制，还需要删除不需要的文件)后上线
* 自动备份
* 统计数据自动入库与初步的统计
* 自动的故障检查与报警，一般来说，需要自己写这样的故障检查脚本：
	* 编写脚本模仿真正的浏览器对重要网页进行静态检查，保证网页上不出现4XX、5XX错误(包括js/css/img以及每个链接)，保证访问每项资源的速度不超过一定时间，以及访问重要页面的时间不超过一定时间，还有静态资源应该设置了Expires以及gzip(主要是HTML/JS/CSS)，以及第三方统计的链接是否正确
	* 使用PhantomJS等headless browser定期自动检查重要网页上的动态信息，特别是iframe以及未知来源的js/css/图像等，一旦发现必然是网页被劫持或者被篡改了，还可以检查是否会在网页上弹出alert、confirm等对话框，之前我们有个网站开发人员在开发的时候加入了alert进行调试，结果直接就上线了，导致每个用户一打开网页就被弹出了一个alert
	* 编写脚本检查第三方网站对我方的链接是否正确，包括友情链接以及搜索引擎搜索链接，不过在做搜索引擎检查时需要注意不要太频繁，不然会被搜索引擎以为是作弊的