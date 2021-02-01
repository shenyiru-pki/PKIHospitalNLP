from UtilsCat import *
import re
import time
from datetime import datetime
import os
import codecs

DataSetRoot = "D:\\DXP\\临床科研分析\\胸外科胸部创伤\\PatientData\\"
OutPutSetRoot = "D:\\DXP\\临床科研分析\\胸外科胸部创伤\\PatientDataOutput\\"

###################################################  整理病例 
bljl,colnames = UsualOpen(os.path.join(DataSetRoot,"胸部创伤Before.txt"),encoding="utf-8-sig",colnames=True)
#colnames=['EMPIID', '就诊流水号', '患者姓名', '文书段落名称', '文本内容']

for idx,item in enumerate(bljl): 
    bljl[idx] = item.replace("\u3000"," ") ##去除全角空格

NewBljl = []
idx=0
tmp=""
while idx < len(bljl)-1:
    if len(bljl[idx].split("\t"))>1 and len(bljl[idx+1].split("\t"))>1:
        if is_number(bljl[idx].split("\t")[1][0:]) and is_number(bljl[idx+1].split("\t")[1][0:]):
            NewBljl.append(bljl[idx])
        elif not is_number(bljl[idx+1].split("\t")[1][0:]):
            tmp+=bljl[idx]
        elif not is_number(bljl[idx].split("\t")[1][0:]) and is_number(bljl[idx+1].split("\t")[1][0:]):
            tmp+=bljl[idx]
            NewBljl.append(tmp)
            tmp = ""
    elif len(bljl[idx].split("\t"))>1 and len(bljl[idx+1].split("\t"))==1:
        tmp+=bljl[idx]
    elif len(bljl[idx].split("\t"))==1 and len(bljl[idx+1].split("\t"))==1:
    	tmp+=bljl[idx]
    elif len(bljl[idx].split("\t"))==1 and len(bljl[idx+1].split("\t"))>1:
        if is_number(bljl[idx+1].split("\t")[1][0:]):
            tmp+=bljl[idx]
            NewBljl.append(tmp)
            tmp = ""
        elif not is_number(bljl[idx+1].split("\t")[1][0:]):
            tmp+=bljl[idx]                
    idx += 1
NewBljl.append(tmp+bljl[-1])

for idx,item in enumerate(NewBljl):
    if len(item.split("\t"))>len(colnames):
        NewBljl[idx] = "\t".join(item.split("\t")[:4])+"\t"+"".join(item.split("\t")[4:])
    elif len(item.split("\t"))==3:
        pass

w = codecs.open(os.path.join(DataSetRoot,"胸部创伤Beforeed.txt"),"w",encoding="utf-8-sig")
w.write("\t".join(colnames)+"\n")
for idx,item in enumerate(NewBljl):
    w.write(item+"\n")
w.close()
