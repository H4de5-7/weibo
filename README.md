# python实现微博热点事件舆情分析（爬虫）

本科时数据挖掘课的大作业，老师说任意选题。正好我们组女生网上冲浪多，就想做个微博舆情分析，选的是这两年争议很大的“错换28年人生”事件。我们对微博获取的舆论进行处理，梳理了事件的脉络，分析了舆论的走向。

整体流程分为五部分：
1、微博爬虫搜集数据（微博、评论等）

2、数据预处理

3、TF-IDF

4、聚类

5、可视化与分析
</br>

由于针对不同情况处理的代码差异会比较大，这里就不放代码了，主要是介绍一下我们的思路，然后把抓取到的85569条数据和停等词字典放在github上。

data.sql “错换28年人生”微博舆论数据

stopw.txt 停等词字典（在百度、腾讯还是哈工大的基础上加了一点点单词）


## 前期准备工作
1、确定分析的事件：

为了方便后续的处理，我们认为搜集到的舆论数据应最少应有1w+条。而我们发现在现实的大部分热点事件中，只有事件被报道的当天会有极大的热度，第二天热度就会急剧下降，下图为一个热点事件随时间的热度变化图（来源为微博的话题指数）。

![话题指数](https://img-blog.csdnimg.cn/90f46131d314478bb90513c0e654e70f.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASDRkZTU=,size_19,color_FFFFFF,t_70,g_se,x_16)

因此我们需要找到一个持续周期长的事件来搜集到足够多的信息，一个组员就想到了”错换28年人生“事件，该事件持续时间长、社会关注度高、也有过一些反转，很适合我们来分析舆论走向。

2、确定分析的事件段：

组员把该事件分为了三个时间段（截止至2021年4月）：

（1）2020年4月-2020年12月26日

（2）2020年12月27日-2021年3月22日

（3）2021年3月23日-2021年4月23日

选择这几个时间节点的原因是：

（1）2020年4月年初该事件第一次被报道出来的时候，引起了人们的高度关注，人们纷纷向姚策捐款。

（2）2021年初的时候一系列关于姚策的负面新闻被爆出来，人们感觉自己被骗了。

（3）2021年3月姚策不幸离世，人们再次热议该事件。


每个时间段中间都有很多相关事件的讨论，yxl同学对热搜条目和时间线做了一份很详尽的整理，我们后续搜集数据也是根据这些时间节点。

![热搜条目](https://img-blog.csdnimg.cn/250e5cf565d84a65ad0c1c5be5f7f83d.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASDRkZTU=,size_20,color_FFFFFF,t_70,g_se,x_16)
![时间线](https://img-blog.csdnimg.cn/5acdf8bfb4db49f8b2d645e86f3028a0.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASDRkZTU=,size_20,color_FFFFFF,t_70,g_se,x_16)

3、确定收集规则：

经过多次讨论后，我们确定搜集五种类型的信息：

（1）热门的微博

（2）热门微博里面的评论（只显示21页）

（3）前50页微博（只显示50页）

（4）前50页微博里面的评论

（5）发微博/评论人的昵称、性别、生日、所在地

我们对上述时间线的所有事件进行三天的信息搜集。

写了两个脚本，一个用来爬取（1）（2）（5），另一个用来爬取（3）（4）（5），重复部分会在数据清洗阶段洗掉。

## 微博爬虫搜集数据
做完前期准备工作后最麻烦的部分来了，要干的事情很多：找接口、测试微博的反爬、爬虫、存数据等。

1、找接口：

微博有三个站点，分别是

移动端 [https://m.weibo.cn/](https://m.weibo.cn/)

电脑端 [https://weibo.cn](https://weibo.cn)

电脑端 [https://weibo.com](https://weibo.com)

我对三个站点都进行了测试，虽然移动端的网页较为简单容易被分析，但是不提供高级搜索，即无法对事件进行精准定位。电脑端的网页均支持高级搜索功能，但是https://weibo.cn 只能按天进行过滤，https://weibo.com 可以按小时进行过滤。考虑到微博设置了每个时间段只能访问最多50页，即50*20=1000条微博。为了收集到尽可能多的数据，我们选择了https://weibo.com 的高级搜索接口[https://s.weibo.com/](https://s.weibo.com/)。

下图对三个站点进行了对比：

![微博的站点](https://img-blog.csdnimg.cn/734559e4888f4a0aa15ff3a52128b5c1.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASDRkZTU=,size_20,color_FFFFFF,t_70,g_se,x_16)

2、测试反爬：

找好了接口后就要测试爬虫了，微博作为国内最大的舆论平台，各项技术也很完善。在网上普遍看见的是同时有封ip和封账户两种反爬机制。如果我们在短时间内使用同一个账号和固定的ip多次访问微博，微博将认定我们有爬虫行为，将封禁账号和ip几分钟，会大大影响爬虫的速度。

但是在实际的测试中（2021年4月，现在不知道什么情况）发现，只要ip不停地变化，账户不会被封禁。因此不需要准备账号池，只需要准备一个ip池，但是我们也不想花钱去整好多代理ip。正好我在网上看到了一个博客https://blog.csdn.net/a1397852386/article/details/111684025 的分享，思路是先爬取免费的匿名代理ip，将几百个代理ip放在一个文件中，然后每次随机选择一个ip访问微博。我测试了下感觉速度还不错。

如果想自动换ip的话，可以用一下红队常用的飞鱼ip，价格大概是40一周，ip都是大陆的。

同时，为了防止微博对相同的User-Agent进行检测，我用UserAgent.random为每个请求生成随机的User-Agent。

3、爬虫：

微博的爬虫很麻烦，各种接口找的头大，用户发表的微博种类很多（视频、链接等），但我们只想要有效的消息，因此还得加一些过滤。有的信息是在xhr里面的，有些是在html里面的。因为微博过于庞大，爬虫的过程中会出现各种奇奇怪怪的问题及报错，报错的时候我就让它跳过该次请求了。用的是XPath，但是后来感觉BeautifulSoup也挺好用的。

本来光爬取微博速度其实很快，但是还要爬取微博中的评论及个人信息（用用户微博id构造url来访问用户的主页获取个人信息），速度顿时就降下来了。爬虫的时候发现爬了一堆重复的评论，结果发现微博设置只显示21页评论，22页及以后的评论与21页的是一样。。给我整无语了，就不能设置只超过21页报错吗（第一轮测试的时候没发现这个问题）。

4、存数据

由于速度跑得有点慢，就让几个队友用她们的电脑一起跑，每个人负责几个事件，跑的数据存在服务器上。最后跑了一个下午加一个晚上（好像）我们得到了85569条原始数据。数据量还是挺满意的。

下图为部分数据：

![image](https://user-images.githubusercontent.com/48757788/156333068-85b87f0f-f396-41da-82c3-d162e472364d.png)

</br>

## 数据预处理
1、数据清洗：

（1）离谱的数据：

上图有些人的出生年龄是2017年、1900年等，这些离谱的生日数据是要被清洗掉的。

（2）无效的数据：

除了空白的文字外，还有一些用户的评论带着转发等字眼，这是微博自动生成的，需要把这些字眼清洗掉。类似的文字还有一些，都需要清洗掉。

（3）重复的数据：

熟悉微博的应该都知道有一些用户在转发或者撰写微博的时候会添加多个标签，这些标签其实是会重复很多次的，因此我们需要清洗掉这些数据。

最后有效数据有62289条。

2、分词：

清洗完数据后需要分词。分词是为了后续的TF-IDF处理。我们使用的是jieba分词工具。

我们把有效的舆论内容分解为多个单词。

3、消除停等词：

有些停等词是体现不出来舆论想表达的意思，只起到连接的作用或者语气助词的作用。如；。，等标点符号，如“从而”“如今”“他”等文字。我在网上找到的停等词库（忘了是百度、腾讯还是哈工大的了）中加了一点点新的单词。

将停等词删除后数据预处理阶段结束。
</br>

## TF-IDF
我们把所有数据按上述的三个时间段分开进行处理。

TF-IDF算法可以反映词语的重要性，网上的理论介绍与代码很多了，这里就不过多介绍了。TF-IDF在这里有两个作用：

1、TF-IDF输出的权重矩阵作为后续聚类的输入。

2、TF-IDF输出每个时间段的热词，这些热词会与后续聚类生成的热词进行比较。
</br>

## 聚类
1、我们直接用的是最经典也是很简单的K-means算法进行聚类。

输入为TF-IDF生成的权重矩阵。

然后我们用轮廓系数作为分类的评判标准。

我们最后选取的是k=8。

2、聚类后将每个簇中权重最高的8个热词生成总的热词。

3、用PCA降维输出聚类的图像。
</br>

## 可视化与分析
1、讨论地区的热力图

![热力图](https://img-blog.csdnimg.cn/71081273b1764f11a8c19e87e99b0e5a.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASDRkZTU=,size_20,color_FFFFFF,t_70,g_se,x_16)

尽管有些数据不可信，但整体东部省份讨论热度更高，北京和广东最高，

2、评论数量随时间的变化

![评论数](https://img-blog.csdnimg.cn/366e33e63eba41a0857aad0415afd115.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASDRkZTU=,size_20,color_FFFFFF,t_70,g_se,x_16)

与我们选择的时间节点类似，2020年4月，该事件第一次被报道出来，引起了人们的关注。2021年初一系列关于姚策的负面新闻被爆出来，大家觉得被姚策骗了。2021年3月姚策不幸离世，人们又开始讨论起来。

3、关注该事件的人群年龄分布饼图

![饼图](https://img-blog.csdnimg.cn/36e6dfb8cbc641aab704db81585eded4.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASDRkZTU=,size_20,color_FFFFFF,t_70,g_se,x_16)

可以看到的是20-30年龄段的人最关心这个事件，也从一定程度上反映了微博用户的年龄分布。

4、2020.04.25-2020.12.26不同观点聚类占比结果

![聚类占比结果](https://img-blog.csdnimg.cn/3d58545143114f99827080937956a8ac.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASDRkZTU=,size_20,color_FFFFFF,t_70,g_se,x_16)

我们选取占比最大的观点3和占比最小的观点6进行分析：

观点3是“孩子,人生,真的,医院,父母,两家,姚策,希望,肝癌,儿子, 亲生,配资,生活,电视剧”。

观点6是“涓滴,之水成,海洋,颗颗,爱心,希望,错换,人生,新生,项目,捐款,成功,携手,凝聚,点燃”。

观点3表明了大家觉得生活就像电影一样戏剧化。

观点6表明了大家想捐钱帮助姚策治病。

5、2020.04.25-2020.12.26的聚类结果图

![聚类结果图](https://img-blog.csdnimg.cn/4b253e66f3e74aa789311e2722fd66fe.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASDRkZTU=,size_20,color_FFFFFF,t_70,g_se,x_16)

由于我们收集到的舆论都是同一个事件的，因此簇的分布比较集中，但是还是能看出来一定的分类结果。

6、TF-IDF提取的2020.04.25-2020.12.26的热词词云

![词云](https://img-blog.csdnimg.cn/21762e32476b41bfb1fd4939b8d2cd6e.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASDRkZTU=,size_20,color_FFFFFF,t_70,g_se,x_16)

7、TF-IDF提取的2020.04.25-2020.12.26热词按权重的排序

![热词的排序一](https://img-blog.csdnimg.cn/d429f6a912664caeb652205b7daf02a0.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASDRkZTU=,size_20,color_FFFFFF,t_70,g_se,x_16)

8、TF-IDF提取的2021.03.23-2021.04.23热词按权重的排序

![热词的排序二](https://img-blog.csdnimg.cn/02c75e0281c1407fac7ae2beb8e18819.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASDRkZTU=,size_20,color_FFFFFF,t_70,g_se,x_16)

可以看到相较于2020.04.25-2020.12.26，人们在这个时间段更关心姚策不幸离世的新闻，同时也希望警察立案调查28年前发生的事情。
</br>

## 总结
由于ddl比较紧，后面的数据分析部分有些简单了，不过通过做这个大作业还是学到了不少东西。













