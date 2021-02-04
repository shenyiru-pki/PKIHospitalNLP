后面NLP需要用到的functions集合
=============================

### import包
```
import time
import re
import numpy as np
import re
import copy
from pyltp import Segmentor ##导入ltp库
import functools
```

### 读取文件
```
def UsualOpen(filename,encoding,colnames):
    with open(filename,encoding=encoding,errors='ignore') as f:
        contents = [line.rstrip("\r\n") for line in f]
    if colnames==True:
        colnames = contents[0].split("\t")
        contents = contents[1:]
        return contents,colnames
    else:
        return contents
```

### 检查每行的长度
```
def RowLengthCheck(contents,colnames): ##检查是否每一行的长度都等于列名
    i = 0
    index_list = []
    for idx,line in enumerate(contents):
        if len(line.split("\t"))!=len(colnames):
            i+=1
            index_list.append(idx)
    print(str(i)+" lines not satisfied")
    if i!=0:
        return index_list
```
### 检查否定词的个数
输入关键词（例如：糖尿病），需要查找的对象（例如：既往史），否定词列表，pattern类型，返回否定词个数和关键词所在的片段。<br>
```
def NegCheck(keyword,sent,NegList,pattern): ##查看关键词phrase中，否定词的个数
    start_idx = sent.index(keyword)
    try:
        e = pattern.search(sent,start_idx).start()
    except:
        e = len(sent)-1
    try:
        d = pattern.search(sent[::-1],(len(sent)-start_idx-len(keyword)+1)).start()
    except:
        d = len(sent)
    evidence = sent[(len(sent)-d):e+1]
    NegNum = ListItemInSent(evidence,NegList)
    return NegNum,evidence
```
### 检查否定词的个数2
有两个关键词，确定两个关键词的否定词的个数</br>
```
def NegCheck2(Keyword1,Keyword2,sent,NegList,pattern): ##查看关键词phrase中(Keyword1...Keyword2)，否定词的个数
    StartIdx = find_all_index_string(sent,Keyword1)
    EndIdx = find_all_index_string(sent,Keyword2)
    CpList = []
    for start_idx in StartIdx:
        for end_idx in EndIdx:
            if end_idx>start_idx:
                CpList.append((start_idx,end_idx))
    minimal = len(sent)
    minimal_cp = CpList[0]
    for cp in CpList:
        if end_idx-start_idx<minimal:
            minimal_cp = cp
    try:
        e = pattern.search(sent,minimal_cp[1]).start()
    except:
        e = len(sent)-1
    try:
        d = pattern.search(sent[::-1],(len(sent)-minimal_cp[0]-len(Keyword1)+1)).start()
    except:
        d = len(sent)
    evidence = sent[(len(sent)-d):e+1] ##切割证据
    NegNum = ListItemInSent(evidence,NegList)
    return NegNum,evidence
```

### 提取时间
```
def ExtractTime(keyword,sent,NegList,pattern,a): ##a为分词class
    NegNumber,evidence = NegCheck(keyword,sent,NegList,pattern)
    a.PrePro(evidence)
    a.SentSplit()
    a.WordSplit()
    a.TimeGet()
    TimeList = []
    if len(a.cand_word)>0:
        for time in a.cand_word:
            if time in evidence:
                TimeList.append(time)
    if len(TimeList)>0:
        for idx,time in enumerate(TimeList):
            if evidence[evidence.index(time)+len(time)]=="前":
                TimeList[idx]=time+"前"
    TimeList+=re.findall(r"(\d{2,4}[./-]\d{1,2}[./-]\d{1,2})",evidence)
    TimeList+=re.findall(r"(\d{2,4}[./-]\d{1,2})",evidence)
    TimeList+=re.findall(r"(\d{2,4}[年.]\d{1,2}月\d{1,2}日)",evidence)
    TimeList+=re.findall(r"(\d{2,4}[年.]\d{1,2}月)",evidence)
    TimeList+=re.findall(r"(\d{2,4}[年])",evidence)
    TimeList = list(set(TimeList))
    if len(TimeList)>0:
        TimeList.sort(key=lambda x:len(x),reverse=True)
        RealTimeList = [TimeList[0]]
        for item in TimeList[1:]:
            for iitem in RealTimeList:
                if item in iitem: ##如果存在2019年9月，则去除2019年
                    pass
                else:
                    RealTimeList.append(item)        
        KeywordIndex = find_all_index_string(evidence,keyword)
        TimeIdx = [(evidence.index(item),evidence.index(item)+len(item)) for item in RealTimeList]
        MinGap = 100
        RealTime = None
        for idx in KeywordIndex: ##找到距离关键词最近的时间
            for iidx in range(0,len(TimeIdx)):
                item = TimeIdx[iidx]
                if item[0]>idx:
                    gap = item[0]-idx
                    if gap < MinGap:
                        MinGap = gap
                        RealTime = RealTimeList[iidx]
                elif item[1]<idx:
                    gap = idx-item[1]
                    if gap < MinGap:
                        MinGap = gap
                        RealTime = RealTimeList[iidx]
        return evidence,RealTime
    else:
        return evidence,None
```
### 提取时间2
仅适用于一句话，且没有标点符号的时候<br>
```
def ExtractTime2(keyword,sent,NegList,a): ##a为分词class,适用于仅一句话并且没有标点符号的情况
    NegNumber=ListItemInSent(sent,NegList)
    evidence = sent
    a.PrePro(evidence)
    a.SentSplit()
    a.WordSplit()
    a.TimeGet()
    TimeList = []
    if len(a.cand_word)>0:
        for time in a.cand_word:
            if time in evidence:
                TimeList.append(time)
    if len(TimeList)>0:
        for idx,time in enumerate(TimeList):
            if evidence.endswith(time):
                pass
            elif evidence[evidence.index(time)+len(time)]=="前":
                TimeList[idx]=time+"前"
    TimeList+=re.findall(r"(\d{2,4}[./-]\d{1,2}[./-]\d{1,2})",evidence)
    TimeList+=re.findall(r"(\d{2,4}[./-]\d{1,2})",evidence)
    TimeList+=re.findall(r"(\d{2,4}[年.]\d{1,2}月\d{1,2}日)",evidence)
    TimeList+=re.findall(r"(\d{2,4}[年.]\d{1,2}月)",evidence)
    TimeList+=re.findall(r"(\d{2,4}[年])",evidence)
    TimeList = list(set(TimeList))
    if len(TimeList)>0:
        TimeList.sort(key=lambda x:len(x),reverse=True)
        RealTimeList = [TimeList[0]]
        for item in TimeList[1:]:
            for iitem in RealTimeList:
                if item in iitem: ##如果存在2019年9月，则去除2019年
                    pass
                else:
                    RealTimeList.append(item)        
        KeywordIndex = find_all_index_string(evidence,keyword)
        TimeIdx = [(evidence.index(item),evidence.index(item)+len(item)) for item in RealTimeList]
        MinGap = 100
        RealTime = None
        for idx in KeywordIndex: ##找到距离关键词最近的时间
            for iidx in range(0,len(TimeIdx)):
                item = TimeIdx[iidx]
                if item[0]>idx:
                    gap = item[0]-idx
                    if gap < MinGap:
                        MinGap = gap
                        RealTime = RealTimeList[iidx]
                elif item[1]<idx:
                    gap = idx-item[1]
                    if gap < MinGap:
                        MinGap = gap
                        RealTime = RealTimeList[iidx]
        return evidence,RealTime
    else:
        return evidence,None
```
### 主诉class
```
class ZSInfoExtraction():
    def __init__(self):
        self.pattern = r',|\.|/|;|\'|`|\[|\]|<|>|\?|:|"|\{|\}|\~|!|@|#|\$|%|\^|&|\(|\)|=|\_|\+|，|。|；|‘|’|【|】|！| |…|（|）'
        self.model_path = "cws.model"
        self.segmentor = Segmentor()
        self.segmentor.load_with_lexicon(self.model_path, 'CliSegCorp.txt')
        self.ChineseDigit = ["一","二","三","四","五","六","七","八","九","十","半","数","两"]
        self.digit = ["1","2","3","4","5","6","7","8","9","0"]
        self.digitall = []
        for item in self.digit:
            if item!="0":
                for itemitem in self.digit:
                    self.digitall.append(item+itemitem)
        self.digitall += self.digit
        unit = ["－","+","-","余","半","年","月","周","日","天","小时","分钟","时"]
        self.unit = []
        for item in unit:
            self.unit.append(item)
            self.unit.append("余"+item)
    def PrePro(self,para):
        newpara = []
        ##去除异常字符
        for idx,word in enumerate(para):
            newword=word
            try:
                word.encode("gbk")
            except UnicodeEncodeError:
                newword=""
            newpara.append(newword)
        self.para = "".join(newpara)
    def SentSplit(self): ##断句
        if "1." in self.para and "2." in self.para: ##将存在：1. 2. 3. 的句子分开
            tmp = self.para.split(";")
            tmp = [a[2:] for a in tmp]
        else:
            tmp = [self.para]
        newtmp = tmp
        for idxa,b in enumerate(tmp):
            newtmp[idxa] = re.split(self.pattern, b) ##把句子按所有标点符号分开
        tmp = [RmFromList(List=item,Element="") for item in newtmp]
        self.sent = tmp ##将段落中的句子分隔开为列表
    def WordSplit(self): ##分词
        self.word = [""]*len(self.sent)
        for idx,item in enumerate(self.sent):
            self.word[idx]=[""]*len(item)
            for iidx,part in enumerate(item):
                tmp = self.segmentor.segment(part)
                self.word[idx][iidx] = " ".join(tmp).split(" ")
        return self.word
    def TimeGet(self):
        allword = functools.reduce(lambda x, y: x.extend(y) or x, [ i if isinstance(i, list) else [i] for i in self.word]) #的搭配主诉中提到的所有part   
        allword = functools.reduce(lambda x, y: x.extend(y) or x, [ i if isinstance(i, list) else [i] for i in allword]) #的搭配主诉中提到的所有part   
        word_idx = [] ##找到所有数字
        for idx,word in enumerate(allword):
            if word.isdigit():               
                word_idx.append((idx,word))
            elif word.startswith(tuple(self.ChineseDigit)):
                word_idx.append((idx,word))
            elif word.startswith(tuple(self.digitall)):
                word_idx.append((idx,word))
        cand_word = [] ##找到所有时间
        for item in word_idx:
            if item[1].endswith(tuple(self.unit)):
                cand_word.append(item[1])
            else:
                idx=item[0]
                tmp = item[1]
                while idx<len(allword)-1: #[10,余，年]:匹配10余年
                    if allword[idx+1] in self.unit:
                        tmp+=allword[idx+1]
                        idx+=1
                    elif allword[idx+1] in self.digitall:
                        tmp+=allword[idx+1]
                        idx+=1
                    elif allword[idx+1] in self.ChineseDigit:
                        tmp+=allword[idx+1]
                        idx+=1
                    elif allword[idx+1].isdigit():
                        tmp+=allword[idx+1]
                        idx+=1
                    else:
                        break
                if tmp.endswith(tuple(self.unit)):
                    cand_word.append(tmp)
        self.cand_word = cand_word
```
### 诊断class 
中间segmentor是可以换列表的，现在是诊断的脑血管疾病列表来拆的<br>
```
class ZDInfoExtraction():
    def __init__(self):
        self.pattern = r',|\.|/|;|\'|`|\[|\]|<|>|\?|:|"|\{|\}|\~|!|@|#|\$|%|\^|&|\(|\)|=|\_|\+|，|。|；|‘|’|【|】|·|！| |…|（|）'
        self.model_path = "cws.model"
        self.segmentor = Segmentor()
        self.segmentor.load_with_lexicon(self.model_path,'脑血管疾病列表.txt')
        self.ChineseDigit = ["一","二","三","四","五","六","七","八","九","十","半","数","两"]
        self.digit = ["1","2","3","4","5","6","7","8","9","0"]
        self.digitall = []
        for item in self.digit:
            if item!="0":
                for itemitem in self.digit:
                    self.digitall.append(item+itemitem)
        self.digitall += self.digit
        unit = ["+","-","余","年","月","周","日","天","小时","分钟"]
        self.unit = []
        for item in unit:
            self.unit.append(item)
            self.unit.append("余"+item)
    def PrePro(self,para):
        newpara = []
        ##去除异常字符
        for idx,word in enumerate(para):
            newword=word
            try:
                word.encode("gbk")
            except UnicodeEncodeError:
                newword=""
            newpara.append(newword)
        self.para = "".join(newpara)
    def SentSplit(self): ##断句
        if "1." in self.para and "2." in self.para: ##将存在：1. 2. 3. 的句子分开
            tmp = self.para.split(";")
            tmp = [a[2:] for a in tmp]
        else:
            tmp = [self.para]
        newtmp = tmp
        for idxa,b in enumerate(tmp):
            newtmp[idxa] = re.split(self.pattern, b) ##把句子按所有标点符号分开
        tmp = [RmFromList(List=item,Element="") for item in newtmp]
        self.sent = tmp ##将段落中的句子分隔开为列表
    def WordSplit(self): ##分词
        self.word = [""]*len(self.sent)
        for idx,item in enumerate(self.sent):
            self.word[idx]=[""]*len(item)
            for iidx,part in enumerate(item):
                tmp = self.segmentor.segment(part)
                self.word[idx][iidx] = " ".join(tmp).split(" ")
        return self.word
    def TimeGet(self):
        allword = functools.reduce(lambda x, y: x.extend(y) or x, [ i if isinstance(i, list) else [i] for i in self.word]) #的搭配主诉中提到的所有part   
        allword = functools.reduce(lambda x, y: x.extend(y) or x, [ i if isinstance(i, list) else [i] for i in allword]) #的搭配主诉中提到的所有part   
        word_idx = [] ##找到所有数字
        for idx,word in enumerate(allword):
            if word.isdigit():               
                word_idx.append((idx,word))
            elif word.startswith(tuple(self.ChineseDigit)):
                word_idx.append((idx,word))
            elif word.startswith(tuple(self.digitall)):
                word_idx.append((idx,word))
        cand_word = [] ##找到所有时间
        for item in word_idx:
            if item[1].endswith(tuple(self.unit)):
                cand_word.append(item[1])
            else:
                idx=item[0]
                tmp = item[1]
                while idx<len(allword)-1: #[10,余，年]:匹配10余年
                    if allword[idx+1] in self.unit:
                        tmp+=allword[idx+1]
                        idx+=1
                    elif allword[idx+1] in self.digitall:
                        tmp+=allword[idx+1]
                        idx+=1
                    elif allword[idx+1] in self.ChineseDigit:
                        tmp+=allword[idx+1]
                        idx+=1
                    else:
                        break
                if tmp.endswith(tuple(self.unit)):
                    cand_word.append(tmp)
        self.cand_word = cand_word
```
### 饮酒class
```
class AlcoholInfoExtraction():
    def __init__(self):
        self.pattern = r',|\.|/|;|\'|`|\[|\]|<|>|\?|:|"|\{|\}|\~|!|@|#|\$|%|\^|&|\(|\)|=|\_|\+|，|。|；|‘|’|【|】|！| |…|（|）'
        self.model_path = "cws.model"
        self.segmentor = Segmentor()
        self.segmentor.load_with_lexicon(self.model_path, 'CliSegCorp.txt')
        self.ChineseDigit = ["一","二","三","四","五","六","七","八","九","十","半","数","两"]
        self.digit = ["1","2","3","4","5","6","7","8","9","0"]
        self.digitall = []
        for item in self.digit:
            if item!="0":
                for itemitem in self.digit:
                    self.digitall.append(item+itemitem)
        self.digitall += self.digit
        unit = ["－","+","-","ml","斤","两","瓶"]
        self.unit = []
        for item in unit:
            self.unit.append(item)
            self.unit.append("余"+item)
    def PrePro(self,para):
        newpara = []
        ##去除异常字符
        for idx,word in enumerate(para):
            newword=word
            try:
                word.encode("gbk")
            except UnicodeEncodeError:
                newword=""
            newpara.append(newword)
        self.para = "".join(newpara)
    def SentSplit(self): ##断句
        if "1." in self.para and "2." in self.para: ##将存在：1. 2. 3. 的句子分开
            tmp = self.para.split(";")
            tmp = [a[2:] for a in tmp]
        else:
            tmp = [self.para]
        newtmp = tmp
        for idxa,b in enumerate(tmp):
            newtmp[idxa] = re.split(self.pattern, b) ##把句子按所有标点符号分开
        tmp = [RmFromList(List=item,Element="") for item in newtmp]
        self.sent = tmp ##将段落中的句子分隔开为列表
    def WordSplit(self): ##分词
        self.word = [""]*len(self.sent)
        for idx,item in enumerate(self.sent):
            self.word[idx]=[""]*len(item)
            for iidx,part in enumerate(item):
                tmp = self.segmentor.segment(part)
                self.word[idx][iidx] = " ".join(tmp).split(" ")
        return self.word
    def AlcoholGet(self):
        allword = functools.reduce(lambda x, y: x.extend(y) or x, [ i if isinstance(i, list) else [i] for i in self.word]) #的搭配主诉中提到的所有part   
        allword = functools.reduce(lambda x, y: x.extend(y) or x, [ i if isinstance(i, list) else [i] for i in allword]) #的搭配主诉中提到的所有part   
        word_idx = [] ##找到所有数字
        for idx,word in enumerate(allword):
            if word.isdigit():               
                word_idx.append((idx,word))
            elif word.startswith(tuple(self.ChineseDigit)):
                word_idx.append((idx,word))
            elif word.startswith(tuple(self.digitall)):
                word_idx.append((idx,word))
        cand_word = [] ##找到所有时间
        for item in word_idx:
            if item[1].endswith(tuple(self.unit)):
                cand_word.append(item[1])
            else:
                idx=item[0]
                tmp = item[1]
                while idx<len(allword)-1: #[10,余，年]:匹配10余年
                    if allword[idx+1] in self.unit:
                        tmp+=allword[idx+1]
                        idx+=1
                    elif allword[idx+1] in self.digitall:
                        tmp+=allword[idx+1]
                        idx+=1
                    elif allword[idx+1] in self.ChineseDigit:
                        tmp+=allword[idx+1]
                        idx+=1
                    elif allword[idx+1].isdigit():
                        tmp+=allword[idx+1]
                        idx+=1
                    else:
                        break
                if tmp.endswith(tuple(self.unit)):
                    cand_word.append(tmp)
        self.cand_word = cand_word
```
