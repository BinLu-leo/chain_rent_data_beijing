# 爬虫实战-链家北京房租数据

本篇是对 `恋习Python` 发布的原创文章[《北京房租大涨？6个维度，数万条数据帮你揭穿》](https://mp.weixin.qq.com/s/vvZ2yBb2eMKP800LUPoAWg%20%E9%93%BE%E5%AE%B6)中涉及的代码部分的解读。

**< 在复现原文代码时，出现了一些报错，在本文中已进行了更改 >**

## 1. 数据获取部分

把目前市场占有率最高的房屋中介公司为目标，来获取北京、上海两大城市的租房信息。
（目标链接：https://bj.lianjia.com/zufang/）

**整体思路是：**

- 先爬取每个区域的url和名称，跟主url拼接成一个完整的url，循环url列表，依次爬取每个区域的租房信息。
- 再爬每个区域的租房信息时，找到最大的页码，遍历页码，依次爬取每一页的二手房信息。

**这里用到的几个爬虫Python包：**

- **requests:** 就是用来请求对链家网进行访问的包
- **lxml:** 解析网页，用xpath表达式与正则表达式一起来获取网页信息，相比bs4速度更快


### 1.1 导入包
```python
import requests
import time
import re
from lxml import etree
```

### 1.2 获取某市区域的所有链接

headers：请求头
content.xpath：

```python
headers = {'User-Agent': 'Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/63.0.3239.108 Safari/537.36'}
```

对第一个正则表达式 `"//dd[@data-index = '0']//div[@class='option-list']/a/text()"` 的解读：

- `//dd` 表示定位到html文本中字段头由 `<dd` 开头。<br>
- `[@data-index = '0']` 表示定位到符合 `data-index = '0'` 这个条件的位置。<br>
- `//div[@class='option-list']/a` 同理向后搜寻，定位到以 `<div` 开头并且符合 `class='option-list` 这个条件的位置。<br>
- `/text()` 表示将上述位置后的文本内容抓取出来。<br>

对第二个正则表达式 `"//dd[@data-index = '0']//div[@class='option-list']/a/@href"` 的解读：

-  同样的代码解读同上。
- `/@href` 表示在上述位置后，将 `href` 后面的内容抓取出来。

```python
# 获取北京市区域的所有链接
def get_areas(url):
    print('start grabing areas')
    headers = {'User-Agent': 'Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/63.0.3239.108 Safari/537.36'}
    response = requests.get(url, headers=headers)
    content = etree.HTML(response.text) # 使用etree来解析html文本
    areas = content.xpath("//dd[@data-index = '0']//div[@class='option-list']/a/text()") # content.xpath来获取北京市的各个辖区
    areas_link = content.xpath("//dd[@data-index = '0']//div[@class='option-list']/a/@href") # content.xpath来获取北京市的各辖区的链接（部分）
    for i in range(1,len(areas)): # 索引0对应的筛选条件是“不限”，即未区分辖区，故跳过
        area = areas[i]
        area_link = areas_link[i]
        link = 'https://bj.lianjia.com' + area_link # 北京市各辖区的第一页
        print(link)
        print("开始抓取页面")
        get_pages(area, link) # 调用get_pages函数
```

> **这个 *get_areas* 函数完成的任务是：**<br>
> 
> - 获取了北京市各区域第一个页面的链接
> - 这个函数的输入是主函数输入的 *url*
> - 最后一个循环中需要调用 *get_pages* 函数
	
### 1.3 通过获取某一区域的页数，来拼接某一页的链接

这个函数嵌套在上一个函数  `get_areas` 中，传入两个变量，一个是北京市的辖区 `area` （str）、另一个是各辖区的首页链接地址 `link` 。

**re.findall(pattern, string)：**以列表形式返回给定模式的所有匹配项

对正则表达式 `page-data=\'{\"totalPage\":(\d+),\"curPage\"` 的解读：

- 以东城区为例，打开东城区第一页的源代码，找到显示该辖区有多少页信息的位置，如下图所示，需要匹配到 “19” ，即是需要的信息。<br>
![链家](https://github.com/lubin007/chain_rent_data_beijing/blob/master/img/lianjia1.png)
- 直接表示出包含 “19” 的这段代码，将目标信息用 `(\d+)` 代替，**\d** 表示0-9的任意一个数字，后面有**+**号说明这个0-9单个数位出现一到多次。
- 此段代码中的 `\'` 、`\"` 中的 **\\** 为转义字符。

```python
#通过获取某一区域的页数，来拼接某一页的链接
def get_pages(area, link):
    headers = {'User-Agent': 'Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/63.0.3239.108 Safari/537.36'}
    resposne = requests.get(link, headers=headers)
    pages =  int(re.findall("page-data=\'{\"totalPage\":(\d+),\"curPage\"", resposne.text)[0]) # 此处的[0]可以省略，因为返回的列表中只有一个参数，表示的就是总页数
    print("这个区域有" + str(pages) + "页")
    for page in range(1,pages+1):
        url = link + 'pg' + str(page) # 此处的link是上个函数的输入，此处的url整个表示各辖区每一页的链接
        print(url)
        print("开始抓取" + str(page) +"的信息")
        get_house_info(area, url) # 调用get_house_info函数
```

### 1.4 获取某一区域某一页的详细房租信息

这个函数嵌套在上一个函数  `get_pages` 中，传入两个变量，一个是北京市的辖区 `area` （str）、另一个是各辖区每一页的链接地址 `url` 。

**time.sleep(secs)：**函数推迟调用线程的运行，可通过参数secs指秒数，表示程序延迟执行的时间。

**try ...	except ... ：**[异常处理：try-except ](https://blog.csdn.net/u012421852/article/details/79277953)将可能出现异常退出的代码用try……except来处理。

对正则表达式 `"//div[@class='where']/span[1]/span/text()"` 中 `"/span[1]/span"` 的解读：

- 首先定位到 `class='where'` 的位置，`/span[1]` 表示再搜索到后面索引值为“1”的span（即第二个span）的位置，`/span/text()` 表示上一个位置后的span后面的文本内容。<br>
![span](https://github.com/lubin007/chain_rent_data_beijing/blob/master/img/lianjia2.png)

对正则表达式 `"([\u4E00-\u9FA5]+)租房"` 的解读：

- `[\u4E00-\u9FA5]` 表示所有汉字的unicode编码范围
-  后面有**+**号说明汉字的数量可以出现一到多次。

对表达式 `with open('链家北京租房.txt','a',encoding='utf-8')` 中参数  `'a'` 的解读：

- 'a' 表示打开一个文件用于追加。如果该文件已存在，文件指针将会放在文件的结尾。也就是说，新的内容将会被写入到已有内容之后。如果该文件不存在，创建新文件进行写入。

**raise e：**

- raise 唯一的一个参数指定了要被抛出的异常。它必须是一个异常的实例或者是异常的类（也就是 Exception 的子类）。
- 如果只想知道这是否抛出了一个异常，并不想去处理它，那么一个简单的 raise 语句就可以再次把它抛出。


```python
#获取某一区域某一页的详细房租信息
def get_house_info(area, url):
    headers = {'User-Agent': 'Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/63.0.3239.108 Safari/537.36'}
    time.sleep(2) # 停顿2秒
    try:
        resposne = requests.get(url, headers=headers)
        content = etree.HTML(resposne.text)
        info=[]
        num = len(content.xpath("//div[@class='where']/a/span/text()")) # 这个页面上有多少条房源的信息
        for i in range(num):
            title = content.xpath("//div[@class='where']/a/span/text()")[i] # 房源信息的标题
            room_type = content.xpath("//div[@class='where']/span[1]/span/text()")[i] # 房源信息的户型
            square = re.findall("(\d+)",content.xpath("//div[@class='where']/span[2]/text()")[i])[0] # 房源信息的面积
            position = content.xpath("//div[@class='where']/span[3]/text()")[i].replace(" ", "") # 房子的朝向
            try:
                detail_place = re.findall("([\u4E00-\u9FA5]+)租房", content.xpath("//div[@class='other']/div/a/text()")[i])[0] # 房子的位置信息
            except Exception as e:
                detail_place = "" # 若出现报错，则此变量为空值
            floor =re.findall("([\u4E00-\u9FA5]+)\(", content.xpath("//div[@class='other']/div/text()[1]")[i])[0] # 房子所在的楼层
            total_floor = re.findall("(\d+)",content.xpath("//div[@class='other']/div/text()[1]")[i])[0] # 房子的总楼层
            try:
                house_year = re.findall("(\d+)",content.xpath("//div[@class='other']/div/text()[2]")[i])[0] # 房子的建楼年限
            except Exception as e:
                house_year = "" # 若出现报错，则此变量为空值
            price = content.xpath("//div[@class='col-3']/div/span/text()")[i] # 房子的租金价格
            with open('链家北京租房.txt','a',encoding='utf-8') as f:
                f.write(area + ',' + title + ',' + room_type + ',' + square + ',' +position+ ','+ detail_place+','+floor+','+total_floor+','+price+','+house_year+'\n')   
            print('writing work has done!continue the next page')
    except Exception as e:
        print(e) # 打印出异常
        #raise e # 抛出异常。在调试时可选，方便定位异常的原因
        time.sleep(30) # 防止被限制，延迟程序30秒
        return get_house_info(area, url) # 再被抛出异常后，重新执行get_house_info函数
```

### 1.5 定义主函数及设置初始输入参数

定义主函数及设置初始输入参数，即我们需要爬取的目标网址。

```python
def main():
    print('start!')
    url = 'https://bj.lianjia.com/zufang'
    get_areas(url)

if __name__ == '__main__':
    main()
```

## 2. 数据清洗预览

> **这里没有进行详细的数据清洗了，详细的步骤请参考公众号之前的文章[Python数据分析过程(基础版)](https://mp.weixin.qq.com/s/lejGoS4Faj7AmXSUuIA-Qg)：**

作者在2018年8月27日爬取下来的数据如下图，共有14177条，10个维度。（因时间关系，没有进一步清洗此数据了。）<br>
![这里写图片描述](https://github.com/lubin007/chain_rent_data_beijing/blob/master/img/lianjia3.png)

在图中，发现 `square` 的最小值为0，明显不合适，通过下面的代码找到对应的数据，之后删除就行了。<br>
![零](https://github.com/lubin007/chain_rent_data_beijing/blob/master/img/lianjia4.png)

## 3. 数据分析可视化

原文章中，没有介绍前期的一些准备工作，对新手来说，实现起来可能会有一些阻碍，在此将其补充一下。

### 3.1 导入包

这里使用的是 `pyecharts` 包，所以需要先导入相应的包。
```python
from pyecharts import Line
from pyecharts import Bar
from pyecharts import Overlap
```

### 3.2 子图一：北京路段_房屋均价分布图

`.agg` ：可以对groupby的结果，同时应用多个函数。
前两行代码是对 `detail_place` 变量进行了分组，然后对各组的 `price` 变量取平均值、计数，存储为DataFrame数据框的形式，。

**Pandas中关于[*set_index*和*reset_index*的用法](https://blog.csdn.net/jingyi130705008/article/details/78162758)：**

- set_index：DataFrame通过set_index方法，可以设置单索引和复合索引。DataFrame.set_index(keys, drop=True, append=False, inplace=False, verify_integrity=False) ；append添加新索引，drop为False，inplace为True时，索引将会还原为列。

- reset_index：reset_index可以还原索引，从新变为默认的整型索引 
DataFrame.reset_index(level=None, drop=False, inplace=False, col_level=0, col_fill=”) ；level控制了具体要还原的那个等级的索引，drop为False则索引列会被还原为普通列，否则会丢失。

`.sort_values` ：按照某一列的大小进行排序。参数ascending为False时，表示降序排列。

```python
#北京路段_房屋均价分布图
detail_place = df.groupby(['detail_place'])
house_com = detail_place['price'].agg(['mean','count'])
house_com.reset_index(inplace=True)
detail_place_main = house_com.sort_values('count',ascending=False)[0:20] # 将counts列按照降序排序，并取出前20位

attr = detail_place_main['detail_place']
v1 = detail_place_main['count']
v2 = detail_place_main['mean']
```
**house_com的输出：**
![12](https://github.com/lubin007/chain_rent_data_beijing/blob/master/img/lianjia5.png)
**detail_place_main的输出：**
![123](https://github.com/lubin007/chain_rent_data_beijing/blob/master/img/lianjia6.png)

**开始[Python 数据可视化](https://www.jianshu.com/p/b718c307a61c)：**

- 第一步：初始化具体类型图表。语法为： 图表名字 =  图表类型（“图的名字”）。
- 第二步：添加图表的数据和设置各种配置项。具体的语法是： 图表类型 `.add()` 。
- 第三步：把图，保存到本地，格式是HTML类型。语法为： 图表类型 `.render()` 。
- 补充：`show_config()` 用于打印输出图表的所有配置项。
- 各参数的解释：
`is_stack` 参数为False时就不堆叠了；
`xaxis_rotate` 横坐标标签的倾斜角度；
`yaxix_min`  ？？这个没有理解，难道指的是yaxis_min，表示y轴的最小值；
`mark_point`  标注点；
-`mark_point_symbol='diamond'` 设置标注点形状；
-`mark_point_textcolor='#40ff27')` 设置标注点颜色；
`mark_line` 标注线；
`xaxis_interval`  x轴坐标标签的间隔；
`mark_point_textcolor`  设置标注点颜色；
`mark_point_symbolsize` 设置标注点大小；
`is_splitline_show`  纵轴的分割线是否显示；
`is_more_utils`  是否展示右侧工具栏。
```python
line = Line("北京主要路段房租均价") # 初始化具体类型图表
line.add("路段",attr,v2,is_stack=True,xaxis_rotate=30,yaxix_min=4.2,mark_point=['min','max'],xaxis_interval=0,line_color='lightblue',line_width=4,mark_point_textcolor='black',mark_point_color='lightblue',is_splitline_show=False)
```
![45](https://github.com/lubin007/chain_rent_data_beijing/blob/master/img/lianjia7.png)

### 3.3 子图二：北京主要路段房屋数量
```python
bar = Bar("北京主要路段房屋数量")
bar.add("路段",attr,v1,is_stack=True,xaxis_rotate=30,yaxix_min=4.2,xaxis_interval=0,is_splitline_show=False)
```
输出结果如下图：
![这里写图片描述](https://github.com/lubin007/chain_rent_data_beijing/blob/master/img/lianjia8.png)

### 3.4 将两个子图结合起来，[Overlap叠加不同类型图表输出](https://blog.csdn.net/qq_42379006/article/details/80841488)

`is_add_yaxis` 表示是否新增一个 y 坐标轴，默认为 False

```python
overlap = Overlap() # 实例化Overlap类
overlap.add(bar) #向overlap中添加图 
overlap.add(line,yaxis_index=1,is_add_yaxis=True)
overlap.render('北京路段_房屋均价分布图.html')
```
![总](https://github.com/lubin007/chain_rent_data_beijing/blob/master/img/lianjia9.png)

**注：没有找到资料解决在Overlap中自定义标题的方案。**

Overlap叠加不同类型图表的 [其他示例](https://blog.csdn.net/MESSI_JAMES/article/details/80850307) 。
