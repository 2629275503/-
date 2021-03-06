# -
某一微博热点事件下方评论的感情分析
# -*- coding: utf-8 -*-
"""
Created on Sat Jun  1 00:06:50 2019

@author: liqx
"""
from copyheaders import headers_raw_to_dict
from bs4 import BeautifulSoup
import requests
import time
import random
from collections import defaultdict
import jieba
import codecs
from wordcloud import WordCloud
from PIL import Image
import matplotlib.pyplot as plt                     #绘制图像的模块
from pylab import *
mpl.rcParams['font.sans-serif'] = ['SimHei']#中文
plt.rcParams['axes.unicode_minus']=False    #解决负数坐标显示问题
from mpl_toolkits.axisartist.axislines import SubplotZero
import numpy as np


######### 爬虫 ##############

headers = b"""
accept:text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8
accept-encoding:gzip, deflate, br
accept-language:zh-CN,zh;q=0.9
cache-control:max-age=0
cookie:_T_WM=1ad1436888fe8b89c7a2863f7986d7b9; SUB=_2A25xpfVRDeRhGeRI6VMT8C3FzT-IHXVTaZsZrDV6PUJbkdAKLWqmkW1NUtUcliAU3-z50ac51gSoCPiLVoblnWq1; SUHB=0wn0NF3PkyCp1I; SCF=Anw_ufU9Bwimf_jyIpvjZPuSWOGgMSGxwWbMWbz6_lrc9w18Uz2_qdur67eB2PdHI7SHFiCLhZYDCtIgm7ehNcc.
upgrade-insecure-requests:1
user-agent:Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/63.0.3239.132 Safari/537.36
"""


# 将请求头字符串转化为字典
headers = headers_raw_to_dict(headers)


# 评论的页数，有多少页，这里就写多少
page = 3000
for i in range(1, page):
    try:
        if i % 50 == 1:
            # 每隔50页打印一下日志
            print("正在抓取第{}页".format(i))
        time.sleep(random.choice(range(10)))
        # 请求网址， 替换网址， 进行爬虫
        url = 'https://weibo.cn/comment/HwokcrhjJ?uid=1784473157&rl=1&page=' + str(i)
        # 发送get请求
        response = requests.get(url=url, headers=headers)
        # 获取评论内容
        html = response.text
        # 利用BeautifulSoup解析网页
        soup = BeautifulSoup(html, 'html.parser')
        # 获取评论信息
        result_1 = soup.find_all(class_='ctt')
        try:
            # 遍历每一页的全部评论
            for j in range(len(result_1)):
                #读取评论信息
                text = result_1[j].get_text().replace(',', '，')
                # 写入csv
                comment = text.split(':')[-1]
                print(comment)
                with open('爬虫结果.txt', 'a+') as f:
                    f.writelines(comment + "\n")
        except:
            continue
    except:
        continue


######### 情感分析 ##############

#使用jieba分词
def div_word(sentence):
    div_list = jieba.cut(sentence)
    div_result = []
    for w in div_list:
        div_result.append(w)
    #读取停用词文件
    stopwords = set()
    fs = codecs.open('停用词.txt', 'r', 'utf-8')
    for word in fs:
        stopwords.add(word.strip())
    fs.close()
    #去除停用词
    return list(filter(lambda x: x not in stopwords, div_result))



#处理情感词文件
def clearBlankLine():
    file1 = open('情感词.txt', 'r', encoding='utf-8') #要去掉空行的文件 
    file2 = open('情感词2.txt', 'w', encoding='utf-8') #生成没有空行的文件
    try:
        for line in file1.readlines():
            if line == '\n':
                line = line.strip("\n")
            file2.write(line)
    finally:
        file1.close()
        file2.close()

clearBlankLine()



#词语分类,找出情感词、否定词、程度副词
def classify_words(word_dict):   
    #读取情感词文件
    sen_file = open('情感词2.txt', 'r+', encoding='utf-8')
    #获取文件内容
    sen_list = sen_file.readlines()
    #创建情感词字典
    sen_dict = defaultdict()
    #转为情感词字典对象，key为情感词，value为情感词对应的分值
    for s in sen_list:
        #每一行内容根据空格分割，索引0是情感词，索引1是情感分值
        #print(s)
        sen_dict[s.split(' ')[0]] = s.split(' ')[1]
 
    #读取否定词文件
    not_word_file = open('否定词.txt', 'r+', encoding='utf-8')
    #创建否定词列表
    not_word_list = not_word_file.readlines()
 
    #读取程度副词文件
    deg_file = open('程度副词.txt', 'r+', encoding='utf-8')
    #获取文件内容
    deg_list = deg_file.readlines()
    #创建程度副词字典
    deg_dic = defaultdict()
    #转为程度副词字典对象，key为程度副词，value为对应的程度值
    for d in deg_list:
        #每一行内容根据空格分割，索引0是情感词，索引1是情感分值        
        deg_dic[d.split(',')[0]] = d.split(',')[1]
 
    #分类结果，词语的index作为key,词语的分值作为value，否定词分值设为-1
    sen_word = dict()
    not_word = dict()
    deg_word = dict()
 
    #分类
    for word in word_dict.keys():
        if word in sen_dict.keys() and word not in not_word_list and word not in deg_dic.keys():
            #找出分词结果中在情感词字典中的词
            sen_word[word_dict[word]] = sen_dict[word]
        elif word in not_word_list and word not in deg_dic.keys():
            #找出分词结果中在否定词列表中的词
            not_word[word_dict[word]] = -1
        elif word in deg_dic.keys():
            #找出分词结果中在程度副词字典中的词
            deg_word[word_dict[word]] = deg_dic[word]
            
    sen_file.close()
    not_word_file.close()
    deg_file.close()

    # 将分类结果返回
    return sen_word, not_word, deg_word

 
#将分词后的列表转为字典，key为单词，value为单词在列表中的索引
def list_to_dict(word_list):
    data = {}
    for x in range(0, len(word_list)):
        data[word_list[x]] = x
    return data
 

#权重
def get_weight(sen_word, not_word, deg_word):
    #权重初始化为1
    W = 1
    #将情感词字典的key转为list
    sen_word_index_list = list(sen_word.keys())
    if len(sen_word_index_list) == 0:
        return W
    #获取第一个情感词的下标，遍历找出程度副词和否定词
    for i in range(0, sen_word_index_list[0]):
        if i in not_word.keys():
            W *= -1
        elif i in deg_word.keys():
            #更新权重，如果有程度副词，分值乘以程度副词的程度分值
            W *= float(deg_word[i])
    return W
 

#计算得分
def socre_sentiment(sen_word, not_word, deg_word, seg_result):
    #权重初始化为1
    W = 1
    score = 0
    #情感词下标初始化
    sentiment_index = -1
    #创建情感词的位置下标集合
    sentiment_index_list = list(sen_word.keys())
    #遍历分词结果
    for i in range(0, len(seg_result)):
        #如果是情感词
        if i in sen_word.keys():
            #权重*情感词得分
            score += W * float(sen_word[i])
            #情感词下标加1，获取下一个情感词的位置
            sentiment_index += 1
            if sentiment_index < len(sentiment_index_list) - 1:
                #判断当前的情感词与下一个情感词之间是否有程度副词或否定词
                for j in range(sentiment_index_list[sentiment_index], sentiment_index_list[sentiment_index + 1]):
                    #更新权重，如果有否定词则取反
                    if j in not_word.keys():
                        W *= -1
                    elif j in deg_word.keys():
                        #更新权重，如果有程度副词则分值乘以程度副词的程度分值
                        W *= float(deg_word[j])
        #定位到下一个情感词
        if sentiment_index < len(sentiment_index_list) - 1:
            i = sentiment_index_list[sentiment_index + 1]
    return score


#计算得分
def setiment_score(sententce):
    #对文档分词
    seg_list = div_word(sententce)
    #将分词结果列表转为dic，然后找出情感词、否定词、程度副词
    sen_word, not_word, deg_word = classify_words(list_to_dict(seg_list))
    #计算得分
    score = socre_sentiment(sen_word, not_word, deg_word, seg_list)
    if score>10:
        return 10
    elif score<-10:
        return -10
    else:
        return score


#分行计算得分
#for line in open("结果.txt"):
#  print(setiment_score(line))
with open(r'情感分析.txt', 'a+') as f:
    for line in open("爬虫结果.txt",'rb'):
        v = str(setiment_score(line))
        f.writelines(v + '\n')



#添加序号
with open(r"情感分析.txt", encoding='UTF-8') as f1:
    f2 = f1.readlines()
    for i in range(0,len(f2)):
        f2[i] = str(i+1) + ' ' + f2[i]
        
with open(r"情感分析2.txt", "w", encoding='UTF-8') as f3:
    f3.writelines(f2)

######### 词云图 ##############

file = open("爬虫结果.txt",'r',encoding='UTF-8')
background_image = np.array(Image.open('中国地图.jpg'))
f=file.read()
text = jieba.cut(f)#结巴分词
stopwords = [line.strip() for line in open('停用词.txt', 'r', encoding='utf-8').readlines()]#停用
outstr = '' # 待返回字符串

for word in text:
   if word not in stopwords:
       outstr += word + " "  #去掉停用词

wc = WordCloud(font_path="C:/Windows/Fonts/simfang.ttf",#字体
                      background_color="white",#背景为白色
                      width=1000,height=880,#宽、高
                      max_words=120,#最大词数
                      mask=background_image,
                      max_font_size=60,#字号
                      min_font_size=8).generate(outstr)
wc.to_file('词云图.jpg')#保存图片
plt.imshow(wc, interpolation="bilinear")
plt.axis("off")
plt.show()
file.close()

######### 变化图 ##############   

fig = plt.figure(1, (10, 6))
ax = SubplotZero(fig, 1, 1, 1)
fig.add_subplot(ax)
ax.axis["xzero"].set_visible(True) #在y=0处添加x轴
#ax.axis["xzero"].label.set_text("")
ax.axis["bottom",'right','top'].set_visible(False) #去掉上、下、右坐标轴
ax.axis["xzero"].set_axisline_style("-|>") #加箭头

filename = '情感分析2.txt'
X,Y = [],[]
with open(filename, 'r') as f:
    lines = f.readlines()
    for line in lines:    
        value = [float(s) for s in line.split()]#分词，写入横纵坐标
        X.append(value[0])
        Y.append(value[1])

plt.ylim(-9,9)   #纵坐标区间    
plt.scatter(X,Y)#画点
plt.ylabel("情感值") #Y轴标签
plt.title("评论情感随时间变化散点图") #标题
plt.show()#显示
fig.savefig('变化.png')#保存散点图
f.close()
