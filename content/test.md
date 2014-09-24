Title: 你好，世界！
Date: 2014-08-22 16:08
Category: Python
Tags: python
Author: Jason
Summary: 你好，世界！
你好，世界，世界，你好。
```Python

#! /usr/bin/python
# -*- coding: utf-8 -*-
import re
import urllib2

def GetHtmlcode(ID):
    # Get the webpage's source html code
    source = 'http://goodinfo.tw/StockInfo/StockDetail.asp?STOCK_ID='
    url = source + ID

    # Header
    headers = { 'User-Agent' : 'Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/31.0.1650.63 Safari/537.36',
                'Accept' : 'text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8',
                'Accept-Charset' : 'Big5,utf-8;q=0.7,*;q=0.3',
                #'Accept-Encoding' : 'gzip,deflate,sdch',
                'Accept-Language' : 'zh-TW,zh;q=0.8,en-US;q=0.6,en;q=0.4,ja;q=0.2' ,
                'Cache-Control' : 'max-age=0',
                'Connection' : 'keep-alive',
                'Cookie' : '427 bytes were stripped',
                'Host' : 'www.goodinfo.tw',
                'Referer' : url }

    # 連到網頁抓取資料
    req= urllib2.Request(url,"",headers)
    response = urllib2.urlopen(req)
    result = response.read().decode('utf-8')
    #print result
    return result

def split_html_tags(tables, list_):
    #print tables.encode('utf-8')
    regex = re.compile("<tr align='center'[\s\S]*?<\/tr>")
    datarow = regex.findall(tables)
    #print len(datarow)
    datarow = datarow[1:]
    str_convert = ''.join(datarow)

    string = str_convert.strip()

    regex = re.compile("<td\s*\S*>(\S+)<\/td>")
    data = regex.findall(str_convert)
    #print data
    #print len(data)

    for i in data:
        list_.append(i)

    #print len(data) / (len(datarow) -1)
    list_.append(len(data) / (len(datarow) -1))

def parse_stock(page):
    regex = re.compile("\<title\>(.*)\<\/title\>")
    title = regex.findall(page)[0]
    title = title[:title.find('-')].strip()
    #print title

    regex = re.compile('<table[\s\S]*?<\/table>')
    datatable = regex.findall(page)
    #print len(datatable)

    data_dict = {}
    Profit_list = []
    Dividends_list = []
    list_ = []

    for l in datatable:
        if l.find(u"\u8fd1&nbsp;10&nbsp;\u5e74&nbsp;\u80a1&nbsp;\u5229&nbsp;\u653f&nbsp;\u7b56") != -1:
            split_html_tags(l, Dividends_list)
        elif l.find(u'\u7372\u3000\u5229\u3000\u72c0\u3000\u6cc1') != -1:
            split_html_tags(l, Profit_list)
    
    data_dict['股票名稱'] = title
    data_dict['股利'] = Dividends_list
    data_dict['績效'] = Profit_list

    #list_.append(Dividends_list)
    #list_.append(Profit_list)
        
    return data_dict

# 計算出股票價值 五年的現金值利率 每股淨值 ROE 毛利率
def pasre_stock_value(dict_):

    print dict_['股票名稱']
    #print dict_['股利']
    #print dict_['績效']
    Dividends_list = []
    Profit_list = []
    Evaluation_four_value_list = []

    list_ = dict_['股利']
    tmp_list = []

    for i in range(0, len(list_) -1):
        tmp_list.append(list_[i])
        if ((i + 1) % list_[-1] == 0):
            Dividends_list.append(tmp_list)
            tmp_list = []


    sum = 0
    # 計算五年股利平均
    for i in range(5):
        sum += float(Dividends_list[i][3])
        #print Dividends_list[i]
        #print Dividends_list[i][3]

    print sum / 5.0


    list_ = dict_['績效']

    for i in range(0, len(list_) -1):
        tmp_list.append(list_[i])
        if ((i + 1) % list_[-1] == 0):
            Profit_list.append(tmp_list)
            tmp_list = []

    for i in Profit_list:
        print i


def main():
    fin = open('StockCode', 'r+')
    StockCodeList = [str(i)for i in fin.read().splitlines()]
    fin.close()

    page = GetHtmlcode('2498')
    dict_ = parse_stock(page)
    pasre_stock_value(dict_)

if __name__ == "__main__":
    main()


```
