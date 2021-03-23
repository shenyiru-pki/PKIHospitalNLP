整理文本之后导出整理好的TXT文件
=================
现在进行的整个流程为：

1.在spotfire中将需要NLP的文本以UTF-8的编码方式txt格式导出到本地(该文本为未整理的文本)，路径为下面的DataSetRoot路径。<br>
2.整理文本后，导出整理后的文本，以txt的格式储存到之前DataSetRoot的路径，到此之前的文本准备阶段就完成了。<br>
3.读取整理后的文本，进行字段抓取，抓取的字段用制表符写成表格的形式以txt形式储存到下面OutPutSetRoot的路径。<br>

### 设置目标路径

这一步建议放最前面比较方便，也不太容易忘记，导入导出的路径在哪<br>
```
DataSetRoot = "D:\\DXP\\PatientData\\"
OutPutSetRoot = "D:\\DXP\\PatientDataOutput\\"
```
###  整理TXT文件成为一行对应一条记录

因为导出成txt文件之后，Python按行读取的时候，可能因为文本中转行的原因，Python会读成两行。也就是一条记录，Python会读成多行。<br>
```
bljl,colnames = UsualOpen(os.path.join(DataSetRoot,"整理前的文本.txt"),encoding="utf-8-sig",colnames=True)
```        
因为要判断新的一条病历记录是从哪儿开始的，所以一定要有一个针对每条记录的唯一编码。例如病案首页号，就诊流水号，报告号。<br>
然后可以把列的名字打出来看看，唯一识别码在第几个，这个index在下面用于识别。在这个例子中唯一识别码是就诊流水号。<br>

colnames=['EMPIID', '就诊流水号', '患者姓名', '文书段落名称', '文本内容']<br>

去除全角空格<br>
```
for idx,item in enumerate(bljl): 
    bljl[idx] = item.replace("\u3000"," ") 
```
根据唯一识别号，来整理一行对应一条记录数据<br>

```
NewBljl = []
idx=0
tmp=""
while idx < len(bljl)-1: #从第一行开始遍历
    if len(bljl[idx].split("\t"))==5 and len(bljl[idx+1].split("\t"))==5: #如果这一行和下一行根据制表符切割之后的元素长度和列名个数一样的话，说明这一行可能是独立的一条记录
        if is_number(bljl[idx].split("\t")[1][0:]) and is_number(bljl[idx+1].split("\t")[1][0:]):
            NewBljl.append(bljl[idx])
        elif not is_number(bljl[idx+1].split("\t")[1][0:]):
            tmp+=bljl[idx]
        elif not is_number(bljl[idx].split("\t")[1][0:]) and is_number(bljl[idx+1].split("\t")[1][0:]):
            tmp+=bljl[idx]
            NewBljl.append(tmp)
            tmp = ""
    elif len(bljl[idx].split("\t"))==5 and len(bljl[idx+1].split("\t"))!=5: #如果这一行根据制表符切割之后的元素长度和列名个数一样的话，而下一行长度不相等，说明下一行可能是这一行中的信息因为换行的原因形成的另起一行，需要合并到这一行来。
        tmp+=bljl[idx]
    elif len(bljl[idx].split("\t"))!=5 and len(bljl[idx+1].split("\t"))!=5: #如果这一行和下一行切割后都和列名个数不相等，说明这一行一定是上一行的信息中的一部分。
    	tmp+=bljl[idx]
    elif len(bljl[idx].split("\t"))!=5 and len(bljl[idx+1].split("\t"))==5: #如果这一行切割后的长度不等于列名个数，而下一行相等的话，说明这一行可能是最后一条需要并到上一条的信息。
        if is_number(bljl[idx+1].split("\t")[1][0:]):
            tmp+=bljl[idx]
            NewBljl.append(tmp)
            tmp = ""
        elif not is_number(bljl[idx+1].split("\t")[1][0:]):
            tmp+=bljl[idx]                
    idx += 1
NewBljl.append(tmp+bljl[-1])
```
对于上述整理完之后，还是有一行对应多条的，继续整合<br>
```
for idx,item in enumerate(NewBljl):
    if len(item.split("\t"))>len(colnames):
        NewBljl[idx] = "\t".join(item.split("\t")[:4])+"\t"+"".join(item.split("\t")[4:]) #切割后index＞4的就直接并到4里面去。
    elif len(item.split("\t"))==5:
        pass
```
可以把最后文本列中不必要的换行符替换成空格‘ ’
```
for idx,item in enumerate(NewBljl):
    text=NewBljl[idx].split('\t')[4]
    text=text.replace('\n',' ')
    NewBljl[idx]=text
```
###  导出整理后的TXT文件
```
w = codecs.open(os.path.join(DataSetRoot,"整理后的文本.txt"),"w",encoding="utf-8-sig")
w.write("\t".join(colnames)+"\n")
for idx,item in enumerate(NewBljl):
    w.write(item+"\n")
w.close()
```
