# python-mem-allocate
基于python的内存分配实验
# <center>操作系统内存分配实验</center>
### <center>邓永盛  2021.2.7</center>

## 1.1 未分配内存区块对象定义


```python
class unused:
    """
    未使用的内存区块对象
    """
    start = int()
    end = int()
    size = int()
    
    def __init__(self,start:int,end:int=None,size:int=None):
        """
        创建一个未分配内存区块对象，end和size必须指定其一
        start: 起始地址
        end: 区块结束地址
        size: 区块大小
        """
        self.start = start
        if end==None:
            self.end = start + size
            self.size = size
        elif size==None:
            self.end = end
            self.size = end - start
        else:
            raise RuntimeError('end，size参数必须指定其一')
            
    def __repr__(self):
        return "(空闲区)\t\t 起始地址:0x%04x\t 结束地址:0x%04x\t 空间大小:0x%04x\n"%(self.start,self.end,self.size)
    
    def __str__(self):
        return self.__repr__()
    
    def allocate(self,allosize:int,process:str):
        print("从空闲区中分配一块0x%04x的空间"%allosize)
        if allosize < self.size:
            #从头部分配空间
            self.start += allosize
            self.size -= allosize
            print("分配成功")
            return used(start=self.start-allosize,process=process,size=allosize)
        else:
            print("该区块空间不足以分配")
```

## 1.2 已分配内存区块对象定义


```python
class used:
    """
    已分配的内存区块对象
    """
    start = int()
    end = int()
    size = int()
    process = str()

    def __init__(self,process:str,start:int,end:int=None,size:int=None):
        """
        创建一个已分配内存区块对象，end和size必须指定其一
        process: 进程名
        start: 起始地址
        end: 区块结束地址
        size: 区块大小
        """
        self.start = start
        self.process = process
        if end==None:
            self.end = start + size
            self.size = size
        elif size==None:
            self.end = end
            self.size = end - start
        else:
            raise RuntimeError('end，size参数必须指定其一')
            
    def __repr__(self):
        return "(已分配区)\t 起始地址:0x%04x\t 结束地址:0x%04x\t 空间大小:0x%04x\t 占用进程:%s\n"%(self.start,self.end,self.size,self.process)
    
    def __str__(self):
        return self.__repr__()
    
    def free(self):
        """
        释放此区块占用的空间，释放成功后返回一个unused对象
        """
        un = unused(self.start,size=self.size)
        print("成功回收 %s 占用的 %d 内存"%(self.process,self.size))
        self.size=0
        self.end=self.start
        return un
```

## 1.3 内存对象定义


```python
class memory:
    """
    内存对象
    """
    size = int()
    freelist = list()
    usedlist = list()
    
    def __init__(self,memorysize:int=0x10000):
        """创建一个内存总大小为memorysize的内存对象"""
        self.size = memorysize
        self.freelist.append(unused(start=0,size=memorysize))
        
    def __repr__(self):
        return "总大小:0x%04x\t 已分配: 0x%04x\t 未分配: 0x%04x\n空闲区:\n%s已分配区:\n%s"%(self.size,self.info()['usedsize'],self.info()['freesize'],"".join(map(str,self.freelist)),"".join(map(str,self.usedlist)))
        
    def __str__(self):
        return self.__repr__()
    
    def allocate(self,process:str,allosize:int,algorithm="FF"):
        """
        从内存中分配allosize大小的连续内存空间
        process: 进程名
        allosize: 需要的空间大小
        """
        if len(list(filter(lambda x: x.size >= allosize,self.freelist))) < 1:
            raise RuntimeError("没有空间大小超过 0x%04x 的连续区块，无法为进程 %s 分配内存空间"%(allosize,process))
        if algorithm == "FF":
            for freemem in filter(lambda x: x.size >= allosize,self.freelist):
                    self.usedlist.append(freemem.allocate(allosize,process))
                    self.collect()
                    break
        elif algorithm == "NF":
            pass
        elif algorithm == "BF":
            pass
        elif algorithm == "WF":
            pass
        else:
            raise RuntimeError("指定的算法错误")
    
    def terminate(self,process:str):
        """结束进程，释放其占用的全部内存"""
        print("结束进程 %s"%process)
        for usedmem in filter(lambda x: x.process==process, self.usedlist):
            self.freelist.append(usedmem.free())
        self.collect()
    
    def collect(self):
        """内存整理函数，删除size为0的区块，合并相邻的空闲区块"""
        #删除size==0的区块
        self.freelist = list(filter(lambda x:x.size>0,self.freelist))
        self.usedlist = list(filter(lambda x:x.size>0,self.usedlist))
        #合并相邻的空闲区块
        for mem1 in self.freelist:
            for mem2 in self.freelist:
                if mem1 is mem2:
                    continue
                if mem1.end == mem2.start:
                    mem1.end = mem2.end
                    mem1.size += mem2.size
                    self.freelist.remove(mem2)
        #将内存区块按照地址排序
        self.freelist.sort(key=lambda x: x.start)
        self.usedlist.sort(key=lambda x: x.start)
    
    def info(self):
        """获取内存的分配情况信息"""
        freesize = sum(map(lambda x: x.size,self.freelist))
        usedsize = sum(map(lambda x: x.size,self.usedlist))
        return {'freesize':freesize,'usedsize':usedsize}
```

## 2.1 创建一个内存对象


```python
mem = memory()
mem
```




    总大小:0x10000	 已分配: 0x0000	 未分配: 0x10000
    空闲区:
    (空闲区)		 起始地址:0x0000	 结束地址:0x10000	 空间大小:0x10000
    已分配区:



## 2.2 为进程p1分配0x100的内存


```python
mem.allocate("p1",0x100)
mem
```

    从空闲区中分配一块0x0100的空间
    分配成功
    




    总大小:0x10000	 已分配: 0x0100	 未分配: 0xff00
    空闲区:
    (空闲区)		 起始地址:0x0100	 结束地址:0x10000	 空间大小:0xff00
    已分配区:
    (已分配区)	 起始地址:0x0000	 结束地址:0x0100	 空间大小:0x0100	 占用进程:p1



## 2.3 为进程p2分配0x200的内存


```python
mem.allocate("p2",0x200)
mem
```

    从空闲区中分配一块0x0200的空间
    分配成功
    




    总大小:0x10000	 已分配: 0x0300	 未分配: 0xfd00
    空闲区:
    (空闲区)		 起始地址:0x0300	 结束地址:0x10000	 空间大小:0xfd00
    已分配区:
    (已分配区)	 起始地址:0x0000	 结束地址:0x0100	 空间大小:0x0100	 占用进程:p1
    (已分配区)	 起始地址:0x0100	 结束地址:0x0300	 空间大小:0x0200	 占用进程:p2



## 2.4 为进程p1再分配0x100的内存


```python
mem.allocate("p1",0x100)
mem
```

    从空闲区中分配一块0x0100的空间
    分配成功
    




    总大小:0x10000	 已分配: 0x0400	 未分配: 0xfc00
    空闲区:
    (空闲区)		 起始地址:0x0400	 结束地址:0x10000	 空间大小:0xfc00
    已分配区:
    (已分配区)	 起始地址:0x0000	 结束地址:0x0100	 空间大小:0x0100	 占用进程:p1
    (已分配区)	 起始地址:0x0100	 结束地址:0x0300	 空间大小:0x0200	 占用进程:p2
    (已分配区)	 起始地址:0x0300	 结束地址:0x0400	 空间大小:0x0100	 占用进程:p1



## 2.5 结束进程p1，释放它占用的所有内存空间


```python
mem.terminate("p1")
mem
```

    结束进程 p1
    成功回收 p1 占用的 256 内存
    成功回收 p1 占用的 256 内存
    




    总大小:0x10000	 已分配: 0x0200	 未分配: 0xfe00
    空闲区:
    (空闲区)		 起始地址:0x0000	 结束地址:0x0100	 空间大小:0x0100
    (空闲区)		 起始地址:0x0300	 结束地址:0x10000	 空间大小:0xfd00
    已分配区:
    (已分配区)	 起始地址:0x0100	 结束地址:0x0300	 空间大小:0x0200	 占用进程:p2



## 2.6 为p3进程分配无法满足的空间大小
这里会按照预期计划报错


```python
mem.allocate("p3",0x10000)
```


    ---------------------------------------------------------------------------

    RuntimeError                              Traceback (most recent call last)

    <ipython-input-9-5a3d665fcc79> in <module>
    ----> 1 mem.allocate("p3",0x10000)
    

    <ipython-input-3-8d9b240af1ba> in allocate(self, process, allosize, algorithm)
         25         """
         26         if len(list(filter(lambda x: x.size >= allosize,self.freelist))) < 1:
    ---> 27             raise RuntimeError("没有空间大小超过 0x%04x 的连续区块，无法为进程 %s 分配内存空间"%(allosize,process))
         28         if algorithm == "FF":
         29             for freemem in filter(lambda x: x.size >= allosize,self.freelist):
    

    RuntimeError: 没有空间大小超过 0x10000 的连续区块，无法为进程 p3 分配内存空间

