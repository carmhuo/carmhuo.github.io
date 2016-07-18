---
layout: post
title: 牛客网用户群体分析
subtitle: 统计分析用户择业方向
author: carm
header-img: img/home-bg.jpg
categories: datamining
tag:
  - python
---

# 牛客网用户群体分析
> 本文属原创，欢迎转载，但请转载时注明出处

最近看到师兄一直在[牛客网](http://www.nowcoder.com/)刷机试题,想想也是，校招马上就要开始了，刷一些经典题型即可以巩固所学知识，也可以对接下来的校招笔试、机试做些准备。[牛客网](http://www.nowcoder.com/)是一个在线编程网站，主要提供互联网公司（如BAT）一些经典面试题的在线编程。突发奇想，既然是为了找工作服务的，是不是注册的用户很大部分是计算机相关专业的在校学生？接着，能不能搜集这些注册用户信息，并分析下这些用户的求职方向？说干就干，接下来就我收集到的信息，进行分析与可视化，并展示统计结果。

> 注：本文写于2016年7月16日，所有数据也是截止2016年7月16日前。

* * *

### 数据收集

我是用python写了一个小爬虫来爬取牛客网用户资料，共收集了近11万用户资料（应该是所有注册用户数了）。

> 注：因为许多用户资料填写不完善，故分析结果可能会有偏差

### 在校与在职

经过数据的预处理，过滤掉一些空置，约1/6（17512个）的用户填写了该信息，统计结果：
<iframe src="/img/niuke_images/pie_school_work.svg" width="760" height="580" ></iframe>
可以看出，牛客网的学生用户还是比较多的。

### 学校Top榜

所有用户中只有约1/4（25327个）的用户填写有学校信息，过滤掉一些脏数据，下面是牛客网的用户所属学校分布信息：
<iframe src="/img/niuke_images/pie_school.svg" width="760" height="580" ></iframe>

### 程序猿与程序媛

约10000多用户填写了性别资料，对这些数据进行统计，结果表明：
<iframe src="/img/niuke_images/pie_gender.svg" width="760" height="580" ></iframe>
男女比例大约为3：1，总的来说还是程序媛比较少。

### 期望的公司
关于在校生所期望的公司分布（数据为10125）：
<iframe src="/img/niuke_images/stu_company.svg" width="760" height="580" ></iframe>
约有50%的学生想去BAT,当然这可能是因为BAT的待遇和发展前景比较好，基本上大部分学生择业方向为各大互联网公司。

### 感兴趣的工作
在校女生统计数据为2037，结果如下：
<iframe src="/img/niuke_images/girls_job.svg" width="760" height="580" ></iframe>
在校男生统计数据为7289：
<iframe src="/img/niuke_images/boys_job.svg" width="760" height="580" ></iframe>
Java工程师与C++工程师可谓是众望所归，除此之外，前端工程师岗位看来也是女生的一个不错选择，男生选择算法和安卓工程师的也比较多。

### 西安电子科大在校生择业方向统计
选取信息资料比较完整的762位西电在校生，统计结果：
<iframe src="/img/niuke_images/xd_job.svg" width="760" height="580" ></iframe>
大方向依然使Java和C++,不过西电的学子还是选择C++工程师的比较多。
<iframe src="/img/niuke_images/xd_company.svg" width="760" height="580" ></iframe>
西电想去华为的人也是比较多的。

### 总结
虽然只是从牛客网上爬下的一些数据，数据量也不大，但也可以管中窥豹。零零散散，总共做了三天，从中感觉到了数据的魅力，自我认为，未来，数据为王！