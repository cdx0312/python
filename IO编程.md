#IO编程

IO在计算机中指Input/Output，也就是输入和输出。由于程序和运行时数据是在内存中驻留，由CPU这个超快的计算机核心来执行，设计到数据交换的地方，通常是磁盘、网络等，就需要IO接口。

比如你打开浏览器，访问新浪首页，浏览器这个程序就需要通过网络IO获取新浪的网页。浏览器首先会发送数据给新浪服务器，告诉它我想要首页的HTML，这个动作是往外发数据，叫Output，随后新浪服务器把网页发过来，这个动作是从外面接收数据，叫Input。所以，通常，程序完成IO操作会有Input和Output两个数据流。当然也有只用一个的情况，比如，从磁盘读取文件到内存，就只有Input操作，反过来，把数据写到磁盘文件里，就只是一个Output操作。

IO编程中，Stream（流）是一个很重要的概念，可以把流想象成一个水管，数据就是水管里的水，但是只能单向流动。Input Stream就是数据从外面（磁盘、网络）流进内存，Output Stream就是数据从内存流到外面去。对于浏览网页来说，浏览器和新浪服务器之间至少需要建立两根水管，才可以既能发数据，又能收数据。

由于CPU和内存的速度远远高于外设的速度，所以，在IO编程中，就存在速度严重不匹配的问题。举个例子来说，比如要把100M的数据写入磁盘，CPU输出100M的数据只需要0.01秒，可是磁盘要接收这100M数据可能需要10秒，怎么办呢？有两种办法：

第一种是CPU等着，也就是程序暂停执行后续代码，等100M的数据在10秒后写入磁盘，再接着往下执行，这种模式称为同步IO；

另一种方法是CPU不等待，只是告诉磁盘，“您老慢慢写，不着急，我接着干别的事去了”，于是，后续代码可以立刻接着执行，这种模式称为异步IO。

同步和异步的区别就在于是否等待IO执行的结果。好比你去麦当劳点餐，你说“来个汉堡”，服务员告诉你，对不起，汉堡要现做，需要等5分钟，于是你站在收银台前面等了5分钟，拿到汉堡再去逛商场，这是同步IO。

你说“来个汉堡”，服务员告诉你，汉堡需要等5分钟，你可以先去逛商场，等做好了，我们再通知你，这样你可以立刻去干别的事情（逛商场），这是异步IO。

很明显，使用异步IO来编写程序性能会远远高于同步IO，但是异步IO的缺点是编程模型复杂。想想看，你得知道什么时候通知你“汉堡做好了”，而通知你的方法也各不相同。如果是服务员跑过来找到你，这是回调模式，如果服务员发短信通知你，你就得不停地检查手机，这是轮询模式。总之，异步IO的复杂度远远高于同步IO。

操作IO的能力都是由操作系统提供的，每一种编程语言都会把操作系统提供的低级C接口封装起来方便使用，Python也不例外。我们后面会详细讨论Python的IO编程接口。

注意，本章的IO编程都是同步模式，异步IO由于复杂度太高，后续涉及到服务器端程序开发时我们再讨论。

##文件读写

读写文件是最常见的IO操作。python内置了读写文件的函数，用法和C是兼容的。

读写文件前，我们必须了解在磁盘读写文件的功能都是由操作系统提供的，现代操作系统不允许普通的程序直接操作磁盘，所以，读写文件就是请求操作系统打开一个文件对象，然后，通过操作系统提供的接口从这个文件对象中读取数据，或者把数据写入这个文件对象。

###读文件

要以读文件的模式打开一个文件对象，使用Python内置的`open()`函数 ，传入文件名和标示符：
```python
>>> f=open('C:/Users/cdxu0/Desktop/open.txt','r')
>>> f.read()
'python open read '
>>> f.close
<built-in method close of _io.TextIOWrapper object at 0x000001A97E9DEB40>
>>>
```
标示符`r`标示读，这样，我们成功打开了一个文件。如果文件不存在，`open()`函数就会抛出一个`IOError`的错误，并且给出错误码和详细的信息高速你文件不存在。
```python
>>> f=open('C:/Users/cdxu0/Desktop/open.py','r')
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
FileNotFoundError: [Errno 2] No such file or directory: 'C:/Users/cdxu0/Desktop/open.py'
```

由于文件读写时都有可能产生`IOError`，一旦出错，后面的`f.close()`就不会调用。所以为了保证无论是否出错都能正确的关闭文件，我们可以使用`try...finally`来实现：
```python
>>> try:
...     f=open('C:/Users/cdxu0/Desktop/test.txt','r')
...     print(f.read())
... finally:
...     if f:
...             f.close()
...
python open read
```

但是每次都这么写比较繁琐，Python引入了`with`语句来自动帮助我们调用`close()`方法。
```python
>>> with open('C:/Users/cdxu0/Desktop/test.txt','r') as f:
...     print(f.read())
...
python open read
```

这和前面的`try...finally`是一样的，但是代码更为简洁，并且不用调用`f.close()`方法。

调用`read()`会一次性读取文件的全部内容，如果文件有10G，内存就爆了，保险起见，要反复调用`read(size)`方法，每次最多读取size个字节的内容。另外，调用`readline()`可以每次读取一行的内容，调用`readlines()`一次读取所有内容并按行返回list。因此，要根据需要决定怎么调用。

如果文件很小，`read()`一次性读取最方便；如果不能确定文件的大小，反复调用`read(size)`比较保险；如果是配置文件，调用`readlines()`最方便。
```python
>>> for line in f.readlines():
...     print(line.strip())
...
python open read
```

###file-like Object

像`open()`函数返回的这种有个`read()`方法的对象，在Python中统称为file-like Object。除了file外，还可以是内存的字节流，网络流，自定义流等等。file-like Object不要求从特定类继承，只要写个`read()`方法就行。

`StringIO`就是在内存中创建的file-like Object，常用作临时缓冲。

###二进制文件

前面讲的默认都是读取文本文件的，并且是UTF-8编码的文本文件。尧都区二进制文件，比如图片、时频等，用`rb`模式打开文件即可：
```python
>>> f=open('C:/Users/cdxu0/Desktop/临时文件/1.png','rb')
>>> f.read()
b'\x89PNG\r\n\x1a\n\x00\x00\x00\rIHDR\x00\x00\x00\x01\x00\x00\x00\x01\x08\x06\x00\x00\x00\x1f\x15\xc4\x89\x00\x00\x00\rIDAT\x08\x99c```\xf8\x0f\x00\x01\x04\x01\x00}\xb2\xc8\xdf\x00\x00\x00\x00I
```

###字符编码

要读取非UTF-8编码的文本文件，需要给`open()`函数传入`encoding`参数，例如，读取GBK编码的文件：
```python
>>> f=open('C:/Users/cdxu0/Desktop/临时文件/test.txt','r',encoding='gbk')
>>> f.read()
'python open read '
```

遇到有些编码不规范的文件，你可能会遇到`UnicodeDecodeError`，因为在文本文件中可能夹杂了一些非法编码的字符。遇到这种情况，`open()`函数还接收一个errors参数，便是如果遇到编码错误后如何处理。最简单的方式是直接忽略。
```
>>> f=open('C:/Users/cdxu0/Desktop/临时文件/test.txt','r',encoding='gbk',errors='ignore')
```

###写文件

写文件和读文件一样 的，唯一的区别在于在调用`open()`函数时，传入标示符`w`或`wb`表示写文本文件或写二进制文件：
```python
>>> f=open('C:/Users/cdxu0/Desktop/临时文件/open.txt','w')
>>> f.write('hehheheh')
8
>>> f.close
<built-in method close of _io.TextIOWrapper object at 0x000002846BDBEB40>
```

你可以反复调用`write()`来写入文件，但是务必要调用`f.close()`来关闭文件。当我们写文件时，操作系统往往不会立刻把文件写入磁盘，而是放到缓存里面，空闲时候再慢慢写入。只有调用`close()`方法时，操作系统才保证把没有写入的数据全部写入磁盘。忘记调`close()`的后果是可能导致数据只写了一部分。还是用with比较保险：
```python
>>> with open('C:/Users/cdxu0/Desktop/临时文件/test.txt','w') as f:
...     f.write('hello,python!')
...
13
```
要写入特定编码的文本文件，请给`open()`函数传入`encoding`参数，将字符串自动转换成指定编码。

--------------------------------------------------
##StringIO和BytesIO

###StringIO

很多时候，数据读写不一定是文件，也可以在内存中读写。StringIO顾名思义就是在内存中读写str。要把str写入StringIO，我们需要先创建一个StringIO，然后像文件一样写入就可以了：
```python
>>> from io import StringIO
>>> f=StringIO()
>>> f.write('hello')
5
>>> f.write(' ')
1
>>> f.write('worls! ')
7
>>> print(f.getvalue())
hello worls!
```

`getvalue()`方法用于获得写入后的str。

要读取StringIO，可以用一个str初始化StringIO，然后像问价一样读取：
```python
>>> from io import StringIO
>>> f=StringIO('hello\nhi\nbye!')
>>> while True:
...     s=f.readline()
...     if s=='':
...             break
...     print(s.strip())
...
hello
hi
bye!
```

###BytesIO

StringIO操作的只能是str，如果要操作二进制数据，就需要使用BytesIO。BytesIO实现了在内存中读写bytes，我们创建一个BytesIO，然后写入一些bytes:
```python
>>> from io import BytesIO
>>> f=BytesIO()
>>> f.write('中文'.encode('utf-8'))
6
>>> print(f.getvalue())
b'\xe4\xb8\xad\xe6\x96\x87'
```

请注意，写入的不是str，而是经过UTF-8编码的bytes。和StringIO类似，可以用一个bytes 初始化BytesIO，然后像读文件一样读取：
```python
>>> from io import BytesIO
>>> f=BytesIO(b'\xe4\xb8\xad\xe6\x96\x87')
>>> f.read()
b'\xe4\xb8\xad\xe6\x96\x87'
```
--------------------------------
##操作文件和目录

如果我们要操作文件、目录，可以在命令行下面输入操作系统提供的各种命令来完成，比如`dir`、`cp`等命令。

如果要在python程序中执行这些目录和文件操作怎么办？其实操作系统提供的命令只是简单的调用了操作系统提供的接口函数，Python内置的`os`模块也可以直接调用操作系统提供的接口函数。

打开python交互式命令行，我们来看看如何使用`os`模块的基本功能：
```python
>>> import os
>>> os.name
'nt'
```

如果是`nt`说明是Windows系统，如果是`posix`，说明是`Linux`、`Unix`、`Mac OS X`系统。要获取详细的系统信息，可以调用`uname()`函数，不过在Windows上不提供。

###环境变量

在操作系统中定义环境变量，全部保存在`os.environ`这个变量中，可以直接查看：
```python
>>> os.environ
environ({'PATH': 'C:\\Windows\\system32;C:\\Windows;C:\\Windows\\System32\\Wbem;C:\\Windows\\System32\\WindowsPowerShell\\v1.0\\;C:\\Users\\cdxu0\\AppData\\Local\\Programs\\Python\\Python35\\Scripts\\;C:\\Users\\cdxu0\\AppData\\Local\\Programs\\Python\\Python35\\', 'PROGRAMFILES(X86)': 'C:\\Program Files (x86)', 'APPDATA': 'C:\\Users\\cdxu0\\AppData\\Roaming', 'FPS_BROWSER_USER_PROFILE_STRING': 'Default', 'COMMONPROGRAMFILES': 'C:\\Program Files\\Common Files', 'ALLUSERSPROFILE': 'C:\\ProgramData', 'COMSPEC': 'C:\\Windows\\system32\\cmd.exe', 'SESSIONNAME': 'Console', 'COMPUTERNAME': 'DESKTOP-F92KHJR', 'USERDOMAIN_ROAMINGPROFILE': 'DESKTOP-F92KHJR', 'SYSTEMDRIVE': 'C:', 'PROCESSOR_LEVEL': '6', 'COMMONPROGRAMFILES(X86)': 'C:\\Program Files (x86)\\Common Files', 'LOGONSERVER': '\\\\MicrosoftAccount', 'PROMPT': '$P$G', 'PROCESSOR_REVISION': '4501', 'PROCESSOR_IDENTIFIER': 'Intel64 Family 6 Model 69 Stepping 1, GenuineIntel', 'PROGRAMFILES': 'C:\\Program Files', 'USERNAME': 'cdxu0', 'PATHEXT': '.COM;.EXE;.BAT;.CMD;.VBS;.VBE;.JS;.JSE;.WSF;.WSH;.MSC', 'WINDIR': 'C:\\Windows', 'USERPROFILE': 'C:\\Users\\cdxu0', 'MOZ_PLUGIN_PATH': 'C:\\PROGRAM FILES (X86)\\FOXIT SOFTWARE\\FOXIT READER PLUS\\plugins\\', 'PROCESSOR_ARCHITECTURE': 'AMD64', 'LOCALAPPDATA': 'C:\\Users\\cdxu0\\AppData\\Local', 'PROGRAMDATA': 'C:\\ProgramData', 'HOMEPATH': '\\Users\\cdxu0', 'PSMODULEPATH': 'C:\\Program Files\\WindowsPowerShell\\Modules;C:\\Windows\\system32\\WindowsPowerShell\\v1.0\\Modules', 'TMP': 'C:\\Users\\cdxu0\\AppData\\Local\\Temp', 'FPS_BROWSER_APP_PROFILE_STRING': 'Internet Explorer', 'OS': 'Windows_NT', 'NUMBER_OF_PROCESSORS': '4', 'PROGRAMW6432': 'C:\\Program Files', 'SYSTEMROOT': 'C:\\Windows', 'COMMONPROGRAMW6432': 'C:\\Program Files\\Common Files', 'ASL.LOG': 'Destination=file', 'PUBLIC': 'C:\\Users\\Public', 'TEMP': 'C:\\Users\\cdxu0\\AppData\\Local\\Temp', 'USERDOMAIN': 'DESKTOP-F92KHJR', 'HOMEDRIVE': 'C:'})
```

要获取某个环境变量的值，可以调用`os.environ.get('key')`：
```python
>>> os.environ.get('PATH')
'C:\\Windows\\system32;C:\\Windows;C:\\Windows\\System32\\Wbem;C:\\Windows\\System32\\WindowsPowerShell\\v1.0\\;C:\\Users\\cdxu0\\AppData\\Local\\Programs\\Python\\Python35\\Scripts\\;C:\\Users\\cdxu0\\AppData\\Local\\Programs\\Python\\Python35\\'
>>> os.environ.get('OS','default')
'Windows_NT'
```

###操作文件和目录

操作文件和目录的函数一部分放在`os`模块中，一部分放在``os.path`模块中，查看、创建、删除目录的调用如下：
```python
>>> os.path.abspath('.')
'C:\\Users\\cdxu0'
>>> os.path.join('C://Users//cdxu0','testdir')
'C://Users//cdxu0\\testdir'
>>> os.mkdir('C://Users/cdxu0/testdir')
>>> os.rmdir('C://Users/cdxu0/testdir')
```
把两个路径合成一个时，不要直接拼字符串，而要通过`os.path,join()`函数，这样可以正确处理不同操作系统的路径分隔符。同样拆分路径时，也不要哦去拆字符串，而要通过`os.path.split()`函数，这样可以把一个路径拆为两部分，后一部分总是最后级别的目录或文件名：
```
>>> os.path.split('C://Users/cdxu0/testdir')
('C://Users/cdxu0', 'testdir')
```
`os.pathsplitext()`可以直接让你得到文件扩展名：
```
>>> os.path.splitext('C://Users/cdxu0/testdir')
('C://Users/cdxu0/testdir', '')
```

这些合并、拆分路径的函数并不要求目录和文件要真实存在，他们只对字符串进行操作。

文件操作使用下面的函数。假定当前目录下有一个`test.txt`文件：
```python
>>> os.mkdir('test.txt')
>>> os.rename('test.txt','test.py')
>>> os.remove('test.py')
```
但是复制文件的函数不在`os`模块中，原因是复制文件并非是由操作系统提供的系统调用。`shutil`模块提供了`copyfile()`的函数，你可以在该模块中找到很多使用函数作为os模块的补充。

最后来看看如何利用python的特性来过滤文件。比如我们要列出当前目录下的所有目录：
```
>>> [x for x in os.listdir('.') if os.path.isdir(x)]
['.android', '.idlerc', '3D Objects', 'AppData', 'Application Data', 'Contacts', 'Cookies', 'Desktop', 'Documents', 'Downloads', 'Evernote', 'Favorites', 'IntelGraphicsProfiles', 'Links', 'Local Settings', 'Music', 'My Documents', 'NetHood', 'OneDrive', 'Pictures', 'PrintHood', 'Recent', 'Saved Games', 'Searches', 'SendTo', 'Templates', 'test.py', 'testdir', 'Videos', '「开始」菜单']
```
要列出所有的，.py文件：

```python
>>> [x for x in os.listdir('.') if os.path.isfile(x) and os.path.splitext(x)[1]=='.py']
['wenzi.py']
```

------------------------------

##序列化

在程序运行过程中，所有的变量都是在内存中，比如，定义一个dict:
```python
 d=dict(name='Bob',age=20,score=99)
```

可以随时修改变量，比如把`name`改为`bill`，但是一旦程序结束，变量所占用的内存就会被系统全部回收。如果没有把修改后的`bill`存储到磁盘上，下次重新运行程序，变量又被初始化为`Bob`。

我们把变量从内存中变为可存储或传输的过程称之为序列化，在Python中叫picking，在其他语言中也被称为serialization，marshalling，flattening等。

序列化之后，就可以把序列化后的内容写入磁盘，或者通过网络传输到别的机器上。反过来吧变量内容从序列化的对象重新读到内存里称之为反序列化，即unpicking。

Python提供了`pickle`模块来实现序列化。

首先，我们尝试把一个对象序列化并写入文件：
```python
>>> import pickle
>>> d=dict(name='Bob',age=20,score=99)
>>> pickle.dumps(d)
b'\x80\x03}q\x00(X\x04\x00\x00\x00nameq\x01X\x03\x00\x00\x00Bobq\x02X\x03\x00\x00\x00ageq\x03K\x14X\x05\x00\x00\x00scoreq\x04Kcu.'
```

`pickle.dumps()`方法把任意对象序列化成一个bytes，然后，就可以把这个bytes写入文件。或者用另一个方法`pickle.dump()`直接把对象序列化后写入一个file-like Object：
```python
>>> f=open('dump.txt','wb')
>>> pickle.dump(d,f)
>>> f.close()
```

看看写入的`dump.txt`文件，内容很乱，这些都是python保存的对象内部信息。当我们要把对象从磁盘读到内存时，可以先把内容读到一个bytes，然后用`pickle.loads()`方法反序列化出对象，也可以直接用`pickle.load()`方法从一个file-like Object中直接反序列化出对象。我们打开另一个Python命令行来反序列化刚才保存的对象：
```python
>>> f=open('dump.txt','rb')
>>> d=pickle.load(f)
>>> f.close
<built-in method close of _io.BufferedReader object at 0x0000028B1BA07938>
>>> d
{'name': 'BOB', 'age': 20, 'score': 99}
```
这个变量和原来的变量是完全不相干的对象，只是内容相同而已。

###JSON

如果我们要在不同的编程语言之间传递对象，就必须把对象序列化为标准格式，比如XMl，但更好的方法是序列化为JSON，因为后者表示出来就是一个字符串，可以被所有语言读取，也可以方便的存储到磁盘或者通过网络传输。JSON不仅是标准格式，并且比XML更快，而且可以直接在WEB页面中读取，非常方便。

JSON表示的对象就是标准的JavaScript语言的对象，JSON和Python内置的数据类型对应如下：

| JSON      |   Pythonv |  
| :-------- | --------:| 
| {}        |   dict   |
|  []       |   list   |
| "string"  |   str    |
|1234.56    |  int或float|
|true/false | True/False|
|null       |None      |

Python内置的json模块提供了非常完善的Python对象到JSON格式的转化。把python对象变为一个JSON：
```python
>>> import json
>>> d=dict(name='bob',age=34,score=99)
>>> json.dumps(d)
'{"score": 99, "name": "bob", "age": 34}'
```

`dumps()`方法返回一个`str`内容就是标准的JSON。类似的，`dump()`方法可以直接把JSON写入一个`file-like Object`。

要把JSON反序列化为Python对象，用`loads()`或者对应的`load()`方法可以直接把JSON的字符串反序列化，后者在`file-like Object`中读取字符串并反序列化：
```python
>>> json_str='{"score": 99, "name": "bob", "age": 34}'
>>> json.loads(json_str)
{'score': 99, 'name': 'bob', 'age': 34}
```

由于JSON标准规定JSON编码是UTF-8，所有我们总是能正确的在python的str和JSON的字符串之间切换。

###JSON进阶

Python的`dict`对象可以直接序列化为JSON的`{}`，不过我们更喜欢用class表示对象，比如定义Student类，然后序列化：
```python
>>> class Student(object):
...     def __init__(self,name,age,score):
...             self.name=name
...             self.age=age
...             self.score=score
...
>>> s=Student('Bob',20,90)
>>> print(json.dumps(s))
```
运行结果是TypeError，原因是Student对象不是一个可以序列化为JSON的对象。查看`dumps()`方法的参数列表，发现除了第一个必须的`obj`参数外，`dumps()`方法还提供了很多可选参数：
[https://docs.python.org/3/library/json.html#json.dumps](https://docs.python.org/3/library/json.html#json.dumps)
这些可选参数就是让我们来定制JSON序列化。前面Student类实例无法序列化为JSON的原因是在默认情况下，`dumps()`方法不知道如何将Student实例变为一个JSON的{}对象。

可选参数`default`就是把任意一个对象变成一个可序列化为JSON的对象，我们只需要为Student专门写一个转换函数，再把函数传进去即可：
```python
>>> def student2dict(std):
...     return {'name':std.name, 'age':std.age, 'score':std.score}
...
>>> print(json.dumps(s,default=student2dict))
{"score": 90, "name": "Bob", "age": 20}
```
这样，Student实例首先被`student2dict()`函数转换成dict，然后再顺利的被序列化为JSON，但是如果改变类，则仍然无法序列化为JSON。我们可以把任意class的实例变为dict：
```python
>>> print(json.dumps(s,default=lambda obj:obj.__dict__))
{"age": 20, "score": 90, "name": "Bob"}
```

因为通常class的实例都有一个`__dict__`属性，它就是一个dict，用来存储实例变量，也有少量例外，如定义了__slots__的class。

同样，如果我们要把JSON反序列化为一个Student 对象实例，`loads()`方法首先转换出一个dict对象，然后我们传入`object_hook`函数负责把一个dict转换为Studnet实例。
```python
>>> def dict2student(d):
...     return Student(d['name'],d['age'],d['score'])
...
>>> json_str='{"age":20, "score":88, "name":"Eason"}'
>>> print(json.loads(json_str, object_hook=dict2student))
<__main__.Student object at 0x00000262C641C6A0>
```
Python语言特定的序列化模块是`pickle`，但是如果要把序列化搞的更通用、更符合Web标准，就可以使用json模块。

json模块的`dumps()`和`loads()`函数是定义的很好的接口的典范。当我们使用时，只需要传入一个必须的参数，但是当默认的序列化或者反序列化机制不满足我们的要求时，我们又可以传入更多参数来定制序列化或者反序列化，即做到了接口简单易用，又做到了充分的扩展性和灵活性。