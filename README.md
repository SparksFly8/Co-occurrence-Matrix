# Python构建共现矩阵并将其三元组表示存储至csv文件
![](https://img.shields.io/badge/license-MIT-000000.svg)
![](https://img.shields.io/cocoapods/dt/AFNetworking.svg)
## 引言：共现矩阵有什么用？

主要用于发现主题，解决**词向量相近关系**的表示； 

将共现矩阵行(列)作为词向量，其表现形式类似于数据结构中图论里学的**邻接矩阵**。在本文中，笔者主要用来统计会议论文作者之间的合作关系。

【举例】：假设有三篇论文，每篇论文作者名字如下。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190405165753278.png)

我们根据上述原始数据构建如下共现矩阵，由如下矩阵可以看出，`Yang Liu`和`Wenwu Zhu`在上述窗口中**共同出现(co-occurrence)过3次**，其实际含义为这两个作者进行过3次合作，**共现的次数越多，我们就认为这两个人的合作关系越紧密**，对应权值也就越高。同理，`Yang Liu`和`Hao Chen`共现过1次、`Wenwu Zhu`和`Hao Chen`共现过2次。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190405154212111.png)

当数据规模很庞大的时候(如2000+个作者)，再用矩阵的形式来表示其关系就**不太合适**，而对于**稀疏矩阵的存储方法**，可以借用数据结构中**三元组**的形式来存储，具体方式如下图：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190405154353672.png)

而Python(其他语言也可)自带的数据结构——**字典**，就可以很方便的将原始数据转换成三元组形式。

## 一、原始数据准备
原始数据按照每篇论文一作、二作依次顺序排列，两两作者之间使用逗号分隔，存储在csv文件中。

本例共有**节点2958个**，**边6040个**。

如何[《将含有特殊字符的xlsx表格数据转化成utf-8编码的csv文件》](https://blog.csdn.net/SL_World/article/details/89034995)请见此。
![在这里插入图片描述](https://img-blog.csdnimg.cn/2019040516163536.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NMX1dvcmxk,size_16,color_FFFFFF,t_70)

## 二、构建共现矩阵并以三元组形式存储到csv文件中
【思路】：遍历每行的作者，对于两两作者A,B**以'A,B'为键**，以**合作频数为值**。对于'B,A'形式的键需反转为'A,B'形式，最后可对字典的值进行降序排序。

##### 1）读取并分割数据

注意此处读取文件的编码方式需设置为`utf-8-sig`，不然得到的列表首个元素会含有`\ufeff`Unicode字符，具体原因百度有。
```js
def get_Co_authors(filePath):
    '''
    读取csv文件获取作者信息并存储到列表中
    :param filePath: csv文件路径
    :return co_authors_list: 一个包含所有作者的列表
    '''
    # 设置编码为utf-8-sig防止首部\ufeff的出现,它是windows系统自带的BOM,用于区分大端和小端UTF-16编码
    with open(filePath, 'r', encoding='utf-8-sig') as f:
        text = f.read()
        co_authors_list = text.split('\n')  # 分割数据中的换行符'\n'两边的数据
        co_authors_list.remove('')          # 删除列表结尾的空字符
        return co_authors_list
```
##### 2）统计节点和关系频数并构建共现矩阵
最终返回两个字典：

**①节点字典**(包含节点名称+词频数)

**②边字典**(包含两两节点关系+关系频数)

构建算法如下，已写好详细注释：
```js
def build_matrix(co_authors_list, is_reverse):
    '''
    根据共同作者列表,构建共现矩阵(存储到字典中),并将该字典按照权值排序
    :param co_authors_list: 共同作者列表
    :param is_reverse: 排序是否倒序
    :return node_str: 三元组形式的节点字符串(且符合csv逗号分隔格式)
    :return edge_str: 三元组形式的边字符串(且符合csv逗号分隔格式)
    '''
    node_dict = {}  # 节点字典,包含节点名+节点权值(频数)
    edge_dict = {}  # 边字典,包含起点+目标点+边权值(频数)
    # 第1层循环,遍历整表的每行作者信息
    for row_authors in co_authors_list:
        row_authors_list = row_authors.split(',') # 依据','分割每行所有作者,存储到列表中
        # 第2层循环,遍历当前行所有作者中每个作者信息
        for index, pre_au in enumerate(row_authors_list): # 使用enumerate()以获取遍历次数index
            # 统计单个作者出现的频次
            if pre_au not in node_dict:
                node_dict[pre_au] = 1
            else:
                node_dict[pre_au] += 1
            # 若遍历到倒数第一个元素,则无需记录关系,结束循环即可
            if pre_au == row_authors_list[-1]:
                break
            connect_list = row_authors_list[index+1:]
            # 第3层循环,遍历当前行该作者后面所有的合作者,以统计两两作者合作的频次
            for next_au in connect_list:
                A, B = pre_au, next_au
                # 固定两两作者的顺序
                if A > B:
                    A, B = B, A
                key = A+','+B  # 格式化为逗号分隔A,B形式,作为字典的键
                # 若该关系不在字典中,则初始化为1,表示作者间的合作次数
                if key not in edge_dict:
                    edge_dict[key] = 1
                else:
                    edge_dict[key] += 1
    # 对得到的字典按照value进行排序
    node_str = sortDictValue(node_dict, is_reverse)  # 节点
    edge_str = sortDictValue(edge_dict, is_reverse)   # 边
    return node_str, edge_str
```

##### 3）对结果字典按照值进行倒序排序

```js
def sortDictValue(dict, is_reverse):
    '''
    将字典按照value排序
    :param dict: 待排序的字典
    :param is_reverse: 是否按照倒序排序
    :return s: 符合csv逗号分隔格式的字符串
    '''
    # 对字典的值进行倒序排序,items()将字典的每个键值对转化为一个元组,key输入的是函数,item[1]表示元组的第二个元素,reverse为真表示倒序
    tups = sorted(dict.items(), key=lambda item: item[1], reverse=is_reverse)
    s = ''
    for tup in tups:  # 合并成csv需要的逗号分隔格式
        s = s + tup[0] + ',' + str(tup[1]) + '\n'
    return s
```

本算法共使用**3次for循环**，时间复杂度为`O(n³)`，共用时`0.63`秒。

执行结果如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190405165920702.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NMX1dvcmxk,size_16,color_FFFFFF,t_70)
##### 4）写入到csv文件

```js
def str2csv(filePath, s):
    '''
    将字符串写入到本地csv文件中
    :param filePath: csv文件路径
    :param s: 待写入字符串(逗号分隔格式)
    '''
    with open(filePath, 'w', encoding='utf-8') as f:
        f.write(s)
    print('写入文件成功,请在'+filePath+'中查看')
```
结果如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190405164310838.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NMX1dvcmxk,size_16,color_FFFFFF,t_70)

【对应我的博客】：https://blog.csdn.net/SL_World/article/details/89034995
