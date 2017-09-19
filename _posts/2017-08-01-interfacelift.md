---
layout: post
#标题配置
title:  批量下载interfacelift的壁纸
#时间配置
date:   2017-08-01 22:34:00 +0800
#大类配置
categories: code
#小类配置
tag: python小应用
---

* content
{:toc}


先上代码

```c
#--coding:UTF-8--
import os
import sys
import time
import urllib
import urllib2
import traceback
from bs4 import BeautifulSoup
def grab_webpage(websites,order):
    content = urllib2.urlopen(websites).read()
    soup=BeautifulSoup(content,'html.parser')
    pictures=soup.find_all(title="Click the preview for more details about this desktop background.")
    picture_list=[]
    for item in pictures:
        if 'previews' in item['src']:
            item_temp=item['src'].replace('previews', '7yz4ma1')
            item_final=item_temp.replace('672x420', '2880x1800')
        else:
            print "item don't have previews"
            item_final=item
        picture_list.append(item_final)
    path=u'/Users/wuwenda/Downloads/壁纸/interfacelift/'
    print "Now start download No."+str(order)+" pictures."
    for i in range(len(picture_list)):
        item_path = path + picture_list[i].split("/")[-1]
        if picture_list[i].split("/")[-1] in os.listdir(path):
            print "File already exists!"
            # sys.stdout.write("File already exists!")
            # time.sleep(3)
            # sys.stdout.write('\t')
            # sys.stdout.write('\r')
            # sys.stdout.flush()
        else:
            try:
                urllib.urlretrieve(picture_list[i], item_path)
            except Exception as e:
                traceback.print_exc()
                print '\tError retrieving the URL:' + picture_list[i]
        print 'Have done NO.'+str(order)+' page: ' + str(i + 1) + '/' + str(len(picture_list))
for i in range(1,390):
    websites='http://interfacelift.com/wallpaper/downloads/date/any/index'+str(i)+'.html'#index1-index389
    grab_webpage(websites,i)
```

Interfacelift上面有好多好看的壁纸，让我看的眼馋啊，但是他们提供批量下载的服务需要79.06刀。

我的个乖乖，贼贵有没有！

作为祖国未来的程序员，我觉得有必要为自己将来的职业技能进行一下修炼（其实就是没钱），决定自己写个程序把壁纸扒下来。

所以让我们开始省钱吧。

### 第一步，获取壁纸URL

所有的壁纸都是从网站上下载的，所以需要知道每一张壁纸的URL，根据URL将壁纸下载到本地。获取URL需要从interfacelift网站上面寻找，每个壁纸有不同的URL。

从网站上寻找URL需要对网页源码进行分析，这里出现了我犯的第一个错误。

我是直接看网页去寻找URL，然后用beautifulsoup将网页源码进行我想要的部分代码的提取。问题出现在，其实显示出来的网页和beautifulsoup拷贝下来的网页是有区别的，区别在于，网站上显示出来的网页源码是动态改变的，随着用户点击的不同，会呈现出不同的样子。而拷贝下来的网页是静态的，是没有做出任何选择的“原味”网页。

因此，我根据网站上看到的源码，写程序在拷贝下来的网页上提取信息的操作没有任何效果，因为两者的不同，我并没有提取到想要的信息。后来发现之后，照着拷贝下来的网页源码进行提取，果断提取到了想要的图片URL。

拷贝网页用到的python库：urllib2

urllib2.urlopen(网页地址).read()这个函数用来将想要的网页拷贝下来，类型应该是string类型的。

之后用到beautifulsoup

BeautifulSoup(扒取的网页内容,’html.parser’)这个函数将“扒取的内容”变成一个BeautifulSoup类，’html.parser’这个参数告诉函数对于“扒取的内容”应该将其当做什么样格式的文档进行处理(html)，BeautiflSoup这个类型类似于一个树形结构，变成这样的类型之后就可以将网页进行逐层分析。

其中用到了find\_all()函数，与find()函数不同的是，find\_all()将所有符合要求的内容全部都找到，find()函数只找到第一个就结束了。BeautifulSoup文档里还有很多有用的函数，值得以后好好看一下。

接下来的问题是，虽然我们获得的网页却是是图片相关的地址，但是这个地址可能不是我们想要的分辨率的壁纸，2880\*1800是mac需要的分辨率，然而获取的地址只有672\*420的大小的预览图片。

值得庆幸的是，所有图片的地址都是有规律的。我们只需要将地址中的部分字段进行替换就可以了。还是so easy的。

### 第二步，下载壁纸

得到地址之后， 开始尝试下载壁纸。

下载壁纸需要使用urllib库。

urllib.urlretrieve(待下载的壁纸，下载到本地的地址)

之后其实想要写一个进度条的程序，暂时还没有做，以后补上好了~

我们的壁纸到这里其实就下载结束了。
```c
# -*- coding: utf-8 -*-
import os
import sys
import time
import urllib
import urllib2
import traceback
from bs4 import BeautifulSoup
def grab_webpage(websites,order):
    content = urllib2.urlopen(websites).read()
    soup=BeautifulSoup(content,'html.parser')
    # //////////////////////////////////////////no.1版本的代码
    pictures=soup.find_all(title="Click the preview for more details about this desktop background.")
    # ............................................................找到壁纸的地址
    picture_resolution = soup.find_all('select')  # ................找到壁纸的可选分辨率
    picture_resolution_list=[]
    for item in picture_resolution:
        tempt=BeautifulSoup(str(item),'html.parser')    # BeautifulSoup函数将str类型变为BeautifulSoup对象，就可以进行操作了
        item_text=tempt.get_text()
        picture_resolution_list.append(item_text)
    # print picture_resolution_list[0].splitlines() 按照"\n"来划分
    # ///////////////////////////////////////////////////////////////////
    resolution_list_real=[]
    for item1 in picture_resolution_list:
        tempt_all=item1.split()  # .................................按空格划分字符串
        order=0
        while(order<len(tempt_all)):
            if(tempt_all[order][0] not in '123456789'):
                tempt_all.remove(tempt_all[order])  # ..................去掉非数字部分
                order=order-1
            order=order+1
        # print tempt_all
        maxium = 0  # ...............................................寻找最大的分辨率
        resolution=''
        for item_number in tempt_all:
            if (('x' in item_number) and (int(item_number[0:item_number.index('x')])>maxium)):
                maxium=int(item_number[0:item_number.index('x')])
                resolution=item_number
        if "2560x1600" in tempt_all:
            resolution="2880x1800"
        resolution_list_real.append(resolution)# ....................将分辨率写到resolution_list_real中
    #print resolution_list_real
    # --------------------------------------------------------------------------find()和find_all()函数的返回值是两个不一样的类型
    # print "///////////////////////////////////////////////////////////////"find()是bs4.element.Tag，可以直接用get_text()函数，
    # print type(soup.find('select'))
    # print type(soup.find_all('select'))
    # ---------------------------------------------------------------------------find_all()是bs4.element.ResultSet，不可以用get_text()

    #print pictures
    # ////////////////////////////////////////////////////////////////////
    picture_list=[]
    for item in range(len(pictures)):
        if 'previews' in pictures[item]['src']:
            item_temp = pictures[item]['src'].replace('previews', '7yz4ma1')
            item_final = item_temp.replace('672x420', resolution_list_real[item])  # ...找到每个图片的下载地址
        else:
            print "item don't have previews"
            item_final=pictures[item]
        picture_list.append(item_final)
    print picture_list
    # ///////////////////////////////////////////////////////////////////
    #print picture_list[1].split("/")
    # ///////////////////////////////////////////////////////////////////////////////////////////////NO.1版本的代码
    path=u'/Users/wuwenda/Downloads/壁纸/interfacelift/'
    print "Now start download No."+str(order)+" pictures."
    for i in range(len(picture_list)):
        item_path = path + picture_list[i].split("/")[-1]
        if picture_list[i].split("/")[-1] in os.listdir(path):
            print "File already exists!"
            # sys.stdout.write("File already exists!")
            # time.sleep(3)
            # sys.stdout.write('\t')
            # sys.stdout.write('\r')
            # sys.stdout.flush()
        else:
            try:
                #print picture_list
                urllib.urlretrieve(picture_list[i], item_path)  # ..........................下载壁纸
            except Exception as e:
                traceback.print_exc()
                print '\tError retrieving the URL:' + picture_list[i]
        print 'Have done NO.'+str(order)+' page: ' + str(i + 1) + '/' + str(len(picture_list))
    # ///////////////////////////////////////////////////////////////////////////////////////////////
# ///////////////////////////////////////////////////////////////////////////////////////////////////////////NO.1版本的代码
for i in range(1,390):
    websites='http://interfacelift.com/wallpaper/downloads/date/any/index'+str(i)+'.html'#index1-index389
    grab_webpage(websites,i)
# ////////////////////////////////////////////////////////////////////////////////////////////////////////////
‘’‘
写在2017-09-19
在运行时候发现一个小问题，并不是所有的壁纸都有2560分辨率的选项，而且2880的分辨率相比好像效果更好。
因此原代码已经不能满足需求了，找不到2560的分辨率，会下载740的低分辨率的壁纸。
这样在原来代码的基础上，我添加了选择最大分辨率的功能，算是对代码新功能的添加。