#### 1.数据链接

爬取数据的链接是：http://map.amap.com/subway/index.html

这个是高德地图对于全国地铁站点的一个可视化界面。

![](https://gitee.com/bai_xiao_fei/picture/raw/master/pic//image-20210509222837340.png)

#### 2.网页分析

既然是可视化那肯定有数据支撑，有数据接口 和 直接显示在页面上。

首先，浏览器打开 F12，定位到上方的城市列表，如图：

![](https://gitee.com/bai_xiao_fei/picture/raw/master/pic//image-20210509223107068.png)

对应的城市列表是直接显示在 div 标签里面的，不过城市是被分成了两部分，一部分在 city-list 里面，一部分在 more-city-list 里面。

而且在每一个城市的 a 标签里面有对应的城市 ID 和城市拼音。

随便点击一个城市，在可视化界面发生变化的同时看到 Network 中出现了一个链接。如图：

![image-20210509223210066](https://gitee.com/bai_xiao_fei/picture/raw/master/pic//image-20210509223210066.png)

链接名称中包含了这个城市的 ID 和拼音，对应的数据就是我们要的地铁站点数据。

不过显然这个数据需要往下稍微深入一点才能发现：

![image-20210509223259408](https://gitee.com/bai_xiao_fei/picture/raw/master/pic//image-20210509223259408.png)

#### 3.思路分析

但是既然有了接口，那获取数据也就很简单的事情

总结一下流程，思路如下：

1. 爬取两个 div 中的城市数据（包括 ID 和拼音），生成城市集合
   遍历城市集合，构造每一个城市的 url
2. 访问 url，爬取对应城市的地铁站点数据
3. 通过地铁站点名去查询其对应所在的城市行政区。

#### 4.获取城市列表

```python
# 获取城市列表
url = 'http://map.amap.com/subway/index.html'
res = requests.get(url, headers={'User-Agent': get_ua()})
res.encoding = res.apparent_encoding
soup = BeautifulSoup(res.text, 'html.parser')

name_dict = []
# 获取显示出的城市列表
for soup_a in soup.find('div', class_='city-list fl').find_all('a'):
    city_name_py = soup_a['cityname']
    city_id = soup_a['id']
    city_name_ch = soup_a.get_text()
    name_dict.append({'name_py': city_name_py, 'id': city_id, 'name_ch': city_name_ch})
# 获取未显示出来的城市列表
for soup_a in soup.find('div', class_='more-city-list').find_all('a'):
    city_name_py = soup_a['cityname']
    city_id = soup_a['id']
    city_name_ch = soup_a.get_text()
    name_dict.append({'name_py': city_name_py, 'id': city_id, 'name_ch': city_name_ch})

df_name = pd.DataFrame(name_dict)
```

```python
# 构造每个城市的url
url = "http://map.amap.com/service/subway?_1818387860087&srhdata=" + id + '_drw_' + cityname + '.json'
```

#### 5.解析城市地铁站点

从 json 中可以很方便的解析每个城市的地铁站点数据

例如：站点所属的地铁线路、站点经纬度等

核心解析代码如下：

```python
# 核心代码
df_per_zd = df_per_zd[['n', 'sl', 'poiid', 'sp']]
df_per_zd['gd经度'] = df_per_zd['sl'].apply(lambda x: x.split(',')[0])
df_per_zd['gd纬度'] = df_per_zd['sl'].apply(lambda x: x.split(',')[1])
df_per_zd.drop('sl', axis=1, inplace=True)
df_per_zd['路线名称'] = data_line['ln']
df_per_zd['城市名称'] = name
```

代码的运行界面如下：

![](https://gitee.com/bai_xiao_fei/picture/raw/master/pic//image-20210509223820414.png)

最终一共是 5001 条数据，对应的全国 40 个开通地铁的城市。

部分数据截图如下：

![QQ截图20210509223927](https://gitee.com/bai_xiao_fei/picture/raw/master/pic//QQ%E6%88%AA%E5%9B%BE20210509223927.png)

#### 爬虫代码

```python
import json
import random
import time

import pandas as pd
import numpy as np
import warnings

import requests
from bs4 import BeautifulSoup


warnings.filterwarnings('ignore')

# 显示所有列
pd.set_option('display.max_columns', None)
# 显示所有行
# pd.set_option('display.max_rows', None)


def get_city_list():
    """
    获取拥有地铁的所有城市
    @return:
    """
    url = 'http://map.amap.com/subway/index.html'
    res = requests.get(url, headers={'User-Agent': 'Mozilla/5.0 (Linux; Android 8.0; Pixel 2 Build/OPD3.170816.012) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/90.0.4430.93 Mobile Safari/537.36'})
    res.encoding = res.apparent_encoding
    soup = BeautifulSoup(res.text, 'html.parser')

    name_dict = []
    # 获取显示出的城市列表
    for soup_a in soup.find('div', class_='city-list fl').find_all('a'):
        city_name_py = soup_a['cityname']
        city_id = soup_a['id']
        city_name_ch = soup_a.get_text()
        name_dict.append({'name_py': city_name_py, 'id': city_id, 'name_ch': city_name_ch})
    # 获取未显示出来的城市列表
    for soup_a in soup.find('div', class_='more-city-list').find_all('a'):
        city_name_py = soup_a['cityname']
        city_id = soup_a['id']
        city_name_ch = soup_a.get_text()
        name_dict.append({'name_py': city_name_py, 'id': city_id, 'name_ch': city_name_ch})

    df_name = pd.DataFrame(name_dict)

    return df_name


def get_metro_info(id, cityname, name):
    """
    地铁线路信息获取
    """
    url = "http://map.amap.com/service/subway?_1618387860087&srhdata=" + id + '_drw_' + cityname + '.json'
    res = requests.get(url, headers={'User-Agent': 'Mozilla/5.0 (Linux; Android 8.0; Pixel 2 Build/OPD3.170816.012) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/90.0.4430.93 Mobile Safari/537.36'})
    data = json.loads(res.text)

    df_data_city = pd.DataFrame()
    if data['l']:
        # 遍历每一条地铁线路
        for data_line in data['l']:
            df_per_zd = pd.DataFrame(data_line['st'])
            df_per_zd = df_per_zd[['n', 'sl', 'poiid', 'sp']]
            df_per_zd['gd经度'] = df_per_zd['sl'].apply(lambda x: x.split(',')[0])
            df_per_zd['gd纬度'] = df_per_zd['sl'].apply(lambda x: x.split(',')[1])
            df_per_zd.drop('sl', axis=1, inplace=True)
            df_per_zd['路线名称'] = data_line['ln']
            df_per_zd['城市名称'] = name

            df_per_zd.rename(columns={'n': '站点名称', 'sp': '拼音名称', 'poiid': 'POI编号'}, inplace=True)
            df_data_city = df_data_city.append(df_per_zd, ignore_index=True)

    return df_data_city


if __name__ == '__main__':
    df_city = pd.DataFrame()
    """获取有地铁站点的城市名"""
    df_name = get_city_list()
    for row_index, data_row in df_name.iterrows():
        print('正在爬取第 {0}/{1} 个城市 {2} 的数据中...'.format(row_index + 1, len(df_name), data_row['name_ch']))
        """遍历每个城市获取地铁站点信息"""
        df_per_city = get_metro_info(data_row['id'], data_row['name_py'], data_row['name_ch'])
        df_city = df_city.append(df_per_city, ignore_index=True)
        # 爬虫休眠
        time.sleep(random.randint(3, 5))

    print(df_city)
    df_city.to_csv(r'全国城市地铁站点信息V202104.csv', encoding='gbk', index=False)
```

