整理导出的TXT文件
=================
### 设置目标路径

这一步建议放最前面比较方便，也不太容易忘记，导入导出的路径在哪<br>
```
DataSetRoot = "D:\\DXP\\临床科研分析\\胸外科胸部创伤\\PatientData\\"
OutPutSetRoot = "D:\\DXP\\临床科研分析\\胸外科胸部创伤\\PatientDataOutput\\"
```
###  整理TXT文件成为一行对应一条记录

因为导出成txt文件之后，Python按行读取的时候，可能因为文本中转行的原因，Python会读成两行。也就是一条记录，Python会读成多行。<br>
```
bljl,colnames = UsualOpen(os.path.join(DataSetRoot,"胸部创伤Before.txt"),encoding="utf-8-sig",colnames=True)
```        
列的顺序要和下面的这个顺序保持一致，不然后面会报错<br>
#colnames=['EMPIID', '就诊流水号', '患者姓名', '文书段落名称', '文本内容']

去除全角空格<br>
```
for idx,item in enumerate(bljl): 
    bljl[idx] = item.replace("\u3000"," ") 
```
根据就诊流水号，来整理一行对应一条记录数据<br>

```
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
```

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
