---
title: slice切片的操作——切片的追加、删除、插入等
date: 2018-07-16 02:02:30
tags: Go
---
# 一、一般操作

1. 声明变量，go自动初始化为nil，长度：0，地址：0，nil

```
func main(){
    var ss []string;
    fmt.Printf("length:%v \taddr:%p \tisnil:%v",len(ss),ss, ss==nil)    
}

---
Running...

length:0     addr:0x0     isnil:true
Success: process exited with code 0.
```

<!-- more -->

2. 切片的追加，删除，插入操作

```
func main(){
    var ss []string;
    fmt.Printf("[ local print ]\t:\t length:%v\taddr:%p\tisnil:%v\n",len(ss),ss, ss==nil)    
    print("func print",ss)
    //切片尾部追加元素append elemnt
    for i:=0;i<10;i++{
        ss=append(ss,fmt.Sprintf("s%d",i));
    }
    fmt.Printf("[ local print ]\t:\tlength:%v\taddr:%p\tisnil:%v\n",len(ss),ss, ss==nil)    
    print("after append",ss)
    //删除切片元素remove element at index
    index:=5;
    ss=append(ss[:index],ss[index+1:]...)
    print("after delete",ss)
    //在切片中间插入元素insert element at index;
    //注意：保存后部剩余元素，必须新建一个临时切片
    rear:=append([]string{},ss[index:]...) 
    ss=append(ss[0:index],"inserted")
    ss=append(ss,rear...)
    print("after insert",ss)
}
func print(msg string,ss []string){
    fmt.Printf("[ %20s ]\t:\tlength:%v\taddr:%p\tisnil:%v\tcontent:%v",msg,len(ss),ss, ss==nil,ss)    
    fmt.Println()
}
------
Running...

[ local print ]    :     length:0    addr:0x0    isnil:true
[           func print ]    :    length:0    addr:0x0    isnil:true    content:[]
[ local print ]    :    length:10    addr:0xc208056000    isnil:false
[         after append ]    :    length:10    addr:0xc208056000    isnil:false    content:[s0 s1 s2 s3 s4 s5 s6 s7 s8 s9]
[         after delete ]    :    length:9    addr:0xc208056000    isnil:false    content:[s0 s1 s2 s3 s4 s6 s7 s8 s9]
[         after insert ]    :    length:10    addr:0xc208056000    isnil:false    content:[s0 s1 s2 s3 s4 inserted s6 s7 s8 s9]

Success: process exited with code 0.
```
3. copy的使用

在使用copy复制切片之前，要保证目标切片有足够的大小，注意是大小，而不是容量，还是看例子：

```
func main() {
    var sa = make ([]string,0);
    for i:=0;i<10;i++{
        sa=append(sa,fmt.Sprintf("%v",i))
        
    }
    var da =make([]string,0,10);
    var cc=0;
    cc= copy(da,sa);
    fmt.Printf("copy to da(len=%d)\t%v\n",len(da),da)
    da = make([]string,5)
    cc=copy(da,sa);
    fmt.Printf("copy to da(len=%d)\tcopied=%d\t%v\n",len(da),cc,da)
     da = make([]string,10)
    cc =copy(da,sa);
    fmt.Printf("copy to da(len=%d)\tcopied=%d\t%v\n",len(da),cc,da)
    
}

---
Running...

copy to da(len=0)    []
copy to da(len=5)    copied=5    [0 1 2 3 4]
copy to da(len=10)    copied=10    [0 1 2 3 4 5 6 7 8 9]
```
从上面运行结果，明显看出，目标切片大小0，容量10，copy不能复制。目标切片大小小于源切片大小，copy就按照目标切片大小复制，不会报错。
# 二、初始大小和容量

当我们使用make初始化切片的时候，必须给出size。go语言的书上一般都会告诉我们，当切片有足够大小的时候，append操作是非常快的。但是当给出初始大小后，我们得到的实际上是一个含有这个size数量切片类型的空元素，看例子：

```
func main(){
    var ss=make([]string,10);
    ss=append(ss,"last");
    print("after append",ss)
    
}
---
Running...

[         after append ]    :    length:11    addr:0xc20804c000    isnil:false    content:[          last]
```
实际上，此时我们应该先用下标为切片元素负值。但是如果我们既想有好的效率，有想继续使用append函数而不想区分是否有空的元素，此时就要请出make的第三个参数，容量，也就是我们通过传递给make，0的大小和足够大的容量数值就行了。

```
func main(){
    var ss=make([]string,0,10);
    ss=append(ss,"last");
    print("after append",ss)
    
}

---
Running...

[         after append ]    :    length:1    addr:0xc20804a000    isnil:false    content:[last]
```
# 三、切片的指针
1. 当我们用append追加元素到切片时，如果容量不够，go就会创建一个新的切片变量，看下面程序的执行结果：

```
func main() {
    var sa []string
fmt.Printf("addr:%p \t\tlen:%v content:%v\n",sa,len(sa),sa);
    for i:=0;i<10;i++{
        sa=append(sa,fmt.Sprintf("%v",i))
        fmt.Printf("addr:%p \t\tlen:%v content:%v\n",sa,len(sa),sa);
    }
    fmt.Printf("addr:%p \t\tlen:%v content:%v\n",sa,len(sa),sa);

}

---
Running ...
addr:0x0         len:0 content:[]
addr:0x1030e0c8         len:1 content:[0]
addr:0x10328120         len:2 content:[0 1]
addr:0x10322180         len:3 content:[0 1 2]
addr:0x10322180         len:4 content:[0 1 2 3]
addr:0x10342080         len:5 content:[0 1 2 3 4]
addr:0x10342080         len:6 content:[0 1 2 3 4 5]
addr:0x10342080         len:7 content:[0 1 2 3 4 5 6]
addr:0x10342080         len:8 content:[0 1 2 3 4 5 6 7]
addr:0x10324a00         len:9 content:[0 1 2 3 4 5 6 7 8]
addr:0x10324a00         len:10 content:[0 1 2 3 4 5 6 7 8 9]
addr:0x10324a00         len:10 content:[0 1 2 3 4 5 6 7 8 9]

//很明显，切片的地址经过了数次改变。
```
2.如果，在make初始化切片的时候给出了足够的容量，append操作不会创建新的切片：

```
func main() {
    var sa = make ([]string,0,10);
fmt.Printf("addr:%p \t\tlen:%v content:%v\n",sa,len(sa),sa);
    for i:=0;i<10;i++{
        sa=append(sa,fmt.Sprintf("%v",i))
        fmt.Printf("addr:%p \t\tlen:%v content:%v\n",sa,len(sa),sa);
    }
    fmt.Printf("addr:%p \t\tlen:%v content:%v\n",sa,len(sa),sa);

}
addr:0x10304140         len:0 content:[]
addr:0x10304140         len:1 content:[0]
addr:0x10304140         len:2 content:[0 1]
addr:0x10304140         len:3 content:[0 1 2]
addr:0x10304140         len:4 content:[0 1 2 3]
addr:0x10304140         len:5 content:[0 1 2 3 4]
addr:0x10304140         len:6 content:[0 1 2 3 4 5]
addr:0x10304140         len:7 content:[0 1 2 3 4 5 6]
addr:0x10304140         len:8 content:[0 1 2 3 4 5 6 7]
addr:0x10304140         len:9 content:[0 1 2 3 4 5 6 7 8]
addr:0x10304140         len:10 content:[0 1 2 3 4 5 6 7 8 9]
addr:0x10304140         len:10 content:[0 1 2 3 4 5 6 7 8 9]

//可见，切片的地址一直保持不变
```
3.如果不能准确预估切片的大小，又不想改变变量（如：为了共享数据的改变），这时候就要请出指针来帮忙了，下面程序中，sa就是osa这个切片的指针，我们共享切片数据和操作切片的时候都使用这个切片地址就ok了，其本质上是：append操作亦然会在需要的时候构造新的切片，不过是将地址都保存到了sa中，因此我们通过该指针始终可以访问到真正的数据。

```
func main() {
    var osa = make ([]string,0);
    sa:=&osa;
    for i:=0;i<10;i++{
        *sa=append(*sa,fmt.Sprintf("%v",i))
        fmt.Printf("addr of osa:%p,\taddr:%p \t content:%v\n",osa,sa,sa);
    }
    fmt.Printf("addr of osa:%p,\taddr:%p \t content:%v\n",osa,sa,sa);
   
}

---
Running...

addr of osa:0xc20800a220,    addr:0xc20801e020      content:&[0]
addr of osa:0xc20801e0a0,    addr:0xc20801e020      content:&[0 1]
addr of osa:0xc20803e0c0,    addr:0xc20801e020      content:&[0 1 2]
addr of osa:0xc20803e0c0,    addr:0xc20801e020      content:&[0 1 2 3]
addr of osa:0xc208050080,    addr:0xc20801e020      content:&[0 1 2 3 4]
addr of osa:0xc208050080,    addr:0xc20801e020      content:&[0 1 2 3 4 5]
addr of osa:0xc208050080,    addr:0xc20801e020      content:&[0 1 2 3 4 5 6]
addr of osa:0xc208050080,    addr:0xc20801e020      content:&[0 1 2 3 4 5 6 7]
addr of osa:0xc208052000,    addr:0xc20801e020      content:&[0 1 2 3 4 5 6 7 8]
addr of osa:0xc208052000,    addr:0xc20801e020      content:&[0 1 2 3 4 5 6 7 8 9]
addr of osa:0xc208052000,    addr:0xc20801e020      content:&[0 1 2 3 4 5 6 7 8 9]
```

