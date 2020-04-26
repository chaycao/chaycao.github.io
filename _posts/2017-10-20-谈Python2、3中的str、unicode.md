---
layout:     post
title:     谈Python2、3中的str、unicode
date:       2017-10-20
author:     ChayCao
header-img: img/post-bg-2015.jpg 
catalog: true
tags:  Python                     
---


## 前言

本文首先对Unicode与UTF-8的区别做一个解释，如果已了解，可跳过该部分。然后会分别对python2，3中的str、unicode进行讲解。有问题的地方，欢迎交流。

## Unicode与UTF-8

- Unicode 是「字符集」
- UTF-8 是「编码规则」


- 字符集：为每一个「字符」分配一个唯一的 ID（学名为码位 / 码点 / Code Point）
- 编码规则：将「码位」转换为字节序列的规则（编码/解码 可以理解为 加密/解密 的过程）

广义的 Unicode 是一个标准，定义了一个字符集以及一系列的编码规则，即 Unicode 字符集和 UTF-8、UTF-16、UTF-32 等等编码……

Unicode 字符集为每一个字符分配一个码位，例如「知」的码位是 30693，记作 U+77E5（30693 的十六进制为 0x77E5）。

```
U+ 0000 ~ U+ 007F: 0XXXXXXX
U+ 0080 ~ U+ 07FF: 110XXXXX 10XXXXXX
U+ 0800 ~ U+ FFFF: 1110XXXX 10XXXXXX 10XXXXXX
U+10000 ~ U+1FFFF: 11110XXX 10XXXXXX 10XXXXXX 10XXXXXX
```
根据上表中的编码规则，之前的「知」字的码位 U+77E5 属于第三行的范围：
```
       7    7    E    5    
    0111 0111 1110 0101    二进制的 77E5
--------------------------
    0111   011111   100101 二进制的 77E5
1110XXXX 10XXXXXX 10XXXXXX 模版（上表第三行）
11100111 10011111 10100101 代入模版
   E   7    9   F    A   5
```

这就是将 U+77E5 按照 UTF-8 编码为字节序列 E79FA5 的过程。

## python2中的str和unicode

###  str与unicode

Python2中：

- str格式本质含义是“某种编码格式”，绝大多数情况下，被引号框起来的字符串，就是str，它本身存储的就是字节码（bytes）。

  ```python
  >>> s = "我爱我的祖国"
  >>> s
  '\xce\xd2\xb0\xae\xce\xd2\xb5\xc4\xd7\xe6\xb9\xfa'
  ```

  那么这个字节码是什么格式的。

  如果这段代码是在解释器上输入的，那么这个s的格式就是解释器的编码格式，对于windows的cmd而言，就是gbk。

  如果将段代码是保存后才执行的，比如存储为utf-8，那么在解释器载入这段程序的时候，就会将s初始化为utf-8编码。

  下面是我在cmd中的尝试

  ```python
  >>> s = "我爱我的祖国"
  >>> print chardet.detect(s)
  {'confidence': 0.99, 'language': 'Chinese', 'encoding': 'GB2312'}
  ```

  在我的测试过程中，有些中文会被识别成别的编码，不是GB2312，如下识别成俄语编码，这个可能跟windows有关。

  ```python
  >>> s = "加油"
  >>> print chardet.detect(s)
  {'confidence': 0.7679697235616183, 'language': 'Russian', 'encoding': 'KOI8-R'}
  >>> s = "中国"
  >>> print chardet.detect(s)
  {'confidence': 0.7679697235616183, 'language': 'Russian', 'encoding': 'IBM855'}
  ```

- unicode类型的含义就是“用unicode编码的字符串”。unicode()是单独的，不像str()是byte类型。

  ```python
  >>> s = u"我爱我的祖国"
  >>> s
  u'\u6211\u7231\u6211\u7684\u7956\u56fd'
  >>> print type(s)
  <type 'unicode'>
  >>> print chardet.detect(s)
  TypeError: Expected object of type bytes or bytearray, got: <type 'unicode'>
  ```

  引号前面的u表示这里创建的是一个Unicode字符串

  Python在进入2.0版后正式定义了了Unicode字符串这个奇怪的特性，目的就是为了处理太多种语言编码的文本。从那时开始，Python语言中的字符串类型就分为两种：一种是传统的Python字符串（各种花样编码），另一种则是新出现的Unicode。

### encode与decode

下面的测试，都将在代码存储为utf-8后，再由解释器执行，也就是说str将初始化为utf-8编码

在python2中的解码（decode）、编码（encode）操作如下：

```python
import chardet
s = "我爱我的祖国"
print type(s)
print chardet.detect(s)
u = s.decode("UTF-8")
print type(u)
s = u.encode("gbk")
print type(s)
print chardet.detect(s)

'''
<type 'str'>
{'confidence': 0.99, 'language': '', 'encoding': 'utf-8'}
<type 'unicode'>
<type 'str'>
{'confidence': 0.99, 'language': 'Chinese', 'encoding': 'GB2312'}
'''
```

从输出可看出，s为*utf-8*编码的字符串，用*decode()*将s解码为unicode对象。

对解码后的unicode对象进行编码，用*encode()*将s编码为*utf-8*对象。

那么，再看下，下面这个情况，s作为字符串，不仅有*decode()*，还有*encode()*。

```python
s = "我爱我的祖国"
print hasattr(s, "decode")
print hasattr(s, "encode")

'''
True
True
'''
```

*decode()*我们是已经使用过了，将某种编码的字符串解码成unicode对象。那么是否能直接将字符串改成另外一种编码呢？这就是str的*encode()*，为我们省去了*decode()*的过程。但这里也有一个坑！

```python
s = "我爱我的祖国"
s = s.encode("gbk")

'''
UnicodeDecodeError: 'ascii' codec can't decode byte 0xce in position 0: ordinal not in range(128)
'''
```

尝试将*utf-8*的str直接变成*gbk*编码，报错了，报错信息翻译如下：

'ascii'编解码器无法解码位置0的字节0xce：序号不在范围（128）

可以看出，程序尝试用*ascii*码对s进行解码，之所以会使用*ascii*，这与系统的默认编码有关：

```python
import sys
print sys.getdefaultencoding()

'''
ascii
'''
```

那么，我们将sys的默认编码设置成*utf-8*，就可以正常运行了

```python
import sys
reload(sys)
sys.setdefaultencoding('utf-8')
s = "我爱我的祖国"
print chardet.detect(s)
s = s.encode("gbk")
print chardet.detect(s)

'''
{'confidence': 0.99, 'language': '', 'encoding': 'utf-8'}
{'confidence': 0.99, 'language': 'Chinese', 'encoding': 'GB2312'}
'''
```

其实*s.encode("gbk")*就相当于

```python
s.decode(defaultencoding),encode('utf-8')
```

## python3中的str和unicode

- str格式的定义变更为”Unicode类型的字符串“，也就是说在默认情况下，被引号框起来的字符串，是使用Unicode编码的。也就是说unicode类型在python3中没有了，python3中的str就相当于python2中的unicode。

  在python3里，str将不再是python2中的字节码，不能作为*chardet.detect*的参数。

  ```python
  import chardet
  s = "我爱我的祖国"
  print (type(s))
  print (chardet.detect(s))

  '''
  <class 'str'>
  TypeError: Expected object of type bytes or bytearray, got: <class 'str'>
  '''
  ```

  str也没有了解码方法*decode()*

  ```python
  s = "我爱我的祖国"
  print (hasattr(s, "decode"))
  print (hasattr(s, "encode"))

  '''
  False
  True
  '''
  ```

  python3中源码文件默认使用utf-8编码，使得以下代码是合法的

  ```python
  >>> 中国 = 'china' 
  >>>print(中国) 
  china
  ```

  下面这个 不知道该怎么解释，不是以unicode编码的形式输出了

  ```python
  >>> s = "我爱我的祖国"
  >>> s
  '我爱我的祖国'
  ```

- 而“不是Unicode的某种编码格式”，比如UTF-8、GBK，这些编码方式被定义为了bytes，这里的bytes和python2中的str有很多相似的地方。

## 参考

1. [知乎（Unicode 和 UTF-8 有何区别？）](https://www.zhihu.com/question/23374078)
2. [知乎（Python2和3中关于str和unicode以及UTF-8的更改到底是什么意思？）](https://www.zhihu.com/question/24891833)
3. [Python中的str与unicode处理方法](http://python.jobbole.com/81244/)
4. [菜鸟教程（字符串）](http://www.runoob.com/python/python-strings.html)
5. [菜鸟教程（Python2.x与3.x版本区别）](http://www.runoob.com/python/python-2x-3x.html)