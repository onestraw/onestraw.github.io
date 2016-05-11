---
layout: single
author_profile: true
comments: true
title: 利用中国气象局API构建天气预报城市代码库
categories: [Python]
tags: [Python]
---
<h1><strong>一、引言</strong></h1>
中国气象局提供了3个开放API，用来查询实时天气数据和预测数据：

1. http://www.weather.com.cn/data/sk/101121005.html

2. http://www.weather.com.cn/data/cityinfo/101121005.html

3. http://m.weather.com.cn/data/101121005.html

这3个网址均以json格式返回数据，第一个和第二个接口返回当天实时的天气数据（它们两个一块用，示例说明如下），第三个返回未来当天和未来五天天气数据。上述3个网址分别返回如下：

1. {“weatherinfo”:{“city”:”定陶”,”cityid”:”101121005″,”temp”:”17″,”WD”:”西南 风”,”WS”:”3 级”,”SD”:”51%”,”WSE”:”3″,”time”:”10:20″,”isRadar”:”0″,”Radar”:”"}}

2. {“weatherinfo”:{“city”:”定 陶”,”cityid”:”101121005″,”temp1″:”24℃”,”temp2″:”11℃”,”weather”:” 晴”,”img1″:”d0.gif”,”img2″:”n0.gif”,”ptime”:”08:00″}}

3. 由于信息太详细了，返回数据比较长，请自行查看。

从上面可以看出，得到一个城市的天气数据非常简单，只要我们有这个城市的代码编号（101121005 是 山东省菏泽市定陶县的代码），就能通过网络获得天气数据。
<h1><strong>二、构建中国城市的代码库</strong></h1>
1.获取省级单位编码API

http://www.weather.com.cn/data/city3jdata/<strong>china.html</strong>

它是一个“字典”类型，包含直辖市和各省级编码，如：

{“10101″:”北京”,”10102″:”上海”,”10103″:”天津”,”10104″:”重庆”,”10105″:”黑龙 江”,”10106″:”吉林”,”10107″:”辽宁”,”10108″:”内蒙古”,”10109″:”河北”,”10110″:”山 西”,”10111″:”陕西”,”10112″:”山东”,….}

2. 根据省级单位编码获取各省的市级单位编码（山东的代码是10112）

http://www.weather.com.cn/data/city3jdata/<strong>provshi/10112.html</strong>

它是一个“字典”类型，包含省级下面的各市级单位编码，如：

{“01″:”济南”,”02″:”青岛”,”03″:”淄博”,”04″:”德州”,”05″:”烟台”,”06″:”潍坊”,”07″:”济 宁”,”08″:”泰安”,”09″:”临沂”,”10″:”菏泽”,”11″:”滨州”,”12″:”东营”,”13″:”威海”,”14″:”枣 庄”,”15″:”日照”,”16″:”莱芜”,”17″:”聊城”}

3. 根据省级单位和市级单位编码获取县级单位编码（山东的代码是10112，菏泽的代码是10）

http://www.weather.com.cn/data/city3jdata/<strong>station/1011201.html</strong>

它是一个“字典”类型，包含市级下面的各县级单位编码，如：

{“01″:”菏泽”,”02″:”鄄城”,”03″:”郓城”,”04″:”东明”,”05″:”定陶”,”06″:”巨野”,”07″:”曹县”,”08″:”成武”,”09″:”单县”}

4. 将上面3级单位代码拼接起来就是查询具体城市天气数据的ID，有两个拼接规则：

<span style="color: #ff0000;"><strong>规则1：省级+市级+县级</strong></span>

<span style="color: #ff0000;"><strong>规则2：省级+县级+市级</strong></span>

<strong>规则2适用于四个直辖市，规则1适用其它省份，我们在构建编码库时，为方便深度优先遍历，全部按规则1进行，在查询具体城市代码时，如果是直辖市，再对编码进行调整。</strong>
<h1><strong>三、Python 实现</strong></h1>
1. 通过urllib模块的urlopen 打开有天气数据的url地址，返回一个socket._fileobject类型的sfo

2. 通过 w = sfo.read()将sfo转化成’str’类型的w

3. w是一个字符串，内容是一个字典，怎样将这个字符串转化成字典呢？

4. 利用python的json模块，d = json.loads(w)
<pre>'''
author:亮剑
site:http://onestraw.net
'''
import urllib
import json
import cPickle
import sys
import pprint

'''
最后得到一个字典cn，可以这样用
cn[u'山东'][u'菏泽'][u'定陶']=101121005
将cn 保存到pk_file文件
'''
def generateCityCode(pk_file):
    basePath = "http://www.weather.com.cn/data/city3jdata/"
    #得到省级代码列表
    china = urllib.urlopen(basePath+"china.html")
    china = json.loads(china.read())
    '''
        {"10101":"北京","10102":"上海","10103":"天津","10104":"重庆","10105":"黑龙江","10106":"吉林","10107":"辽宁",
        "10108":"内蒙古","10109":"河北","10110":"山西","10111":"陕西","10112":"山东","10113":"新疆","10114":"西藏",
        "10115":"青海","10116":"甘肃","10117":"宁夏","10118":"河南","10119":"江苏","10120":"湖北","10121":"浙江",
        "10122":"安徽","10123":"福建","10124":"江西","10125":"湖南","10126":"贵州","10127":"四川","10128":"广东",
        "10129":"云南","10130":"广西","10131":"海南","10132":"香港","10133":"澳门","10134":"台湾"}
        china是上面这样的一个字典，将它反转过来，即省份作为key，编码作为value
    '''
    cn={}
    for k in china.keys():
        cn[china[k]] = k
    for k in cn.keys():
        province = urllib.urlopen(basePath+"provshi/"+cn[k]+".html")
        province = json.loads(province.read())
        '''
        根据  cn[u'山东']=10112  查询得到市级编码
        {"01":"济南","02":"青岛","03":"淄博","04":"德州","05":"烟台","06":"潍坊","07":"济宁","08":"泰安","09":"临沂",
        "10":"菏泽","11":"滨州","12":"东营","13":"威海","14":"枣庄","15":"日照","16":"莱芜","17":"聊城"}
        下面同样将字典逆转过来,并拼接上省级代码
        '''
        prov = {}
        for sk in province.keys():
            prov[province[sk]] = cn[k]+sk   
        
        for sk in prov.keys():
            city = urllib.urlopen(basePath+"station/"+prov[sk]+".html")
            city = json.loads(city.read())
            '''
            根据prov[u'菏泽']=1011210 等到县级的编码
            {"01":"菏泽","02":"鄄城","03":"郓城","04":"东明","05":"定陶","06":"巨野","07":"曹县","08":"成武","09":"单县"}
            '''
            cities = {}
            for ssk in city.keys():
                cities[city[ssk]] = prov[sk]+ssk
            prov[sk] = cities
        cn[k] = prov
    #将三级字典存储到pk文件
    fp = open(pk_file,'wb')
    cPickle.dump(cn,fp)
    fp.close()
    #return cn
'''
根据&lt;省、市、区/县&gt;进行天气查询
'''
def queryWeather(sheng,shi,xian):
    weatherBasePath = "http://www.weather.com.cn/data/sk/"
    fp = open('citycode.pk','rb')
    china = cPickle.load(fp)
    fp.close()
    code = china[sheng][shi][xian]
    if not code:
        sys.exit(1)
    #四个直辖市，需要将code中的市级代码和区级代码交换位置
    fourCity = [u'北京',u'上海',u'天津',u'重庆']
    if sheng in fourCity:
        cl = len(code)
        adjust_code = code[0:cl-4]+code[cl-2:cl]+code[cl-4:cl-2]
        code = adjust_code
    w = urllib.urlopen(weatherBasePath+code+".html")
    w = json.loads(w.read())
    #w是一个字典，此处的天气状况的解析省去
    pprint.pprint(w)

if __name__=='__main__':
    #generateCityCode('citycode.pk')
    queryWeather(u'山东',u'菏泽',u'定陶')
    queryWeather(u'北京',u'北京',u'海淀')
    queryWeather(u'上海',u'上海',u'浦东')
    queryWeather(u'天津',u'天津',u'塘沽')
    queryWeather(u'重庆',u'重庆',u'江津')
    
</pre>
附：

<em>API接口（6个网址）参考：http://www.cnblogs.com/laosan/p/how-to-write-a-weather-api.html</em>

但是这篇文章没有指出直辖市的代码特殊性。
