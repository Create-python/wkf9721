目录：

- \1. 安装pyinstaller
- \2. 打包初体验
- \3. 打包进阶体验
- \4. 带配置文件打包（高级）
- \5. 添加隐式调用库（高级）

## 1. 安装pyinstaller

`PyInstaller`是一个用来将`Python`程序打包成一个独立可执行文件的第三方包。

因是第三方包，所以需要安装一下：

```
pip install pyinstaller
```

或者升级到最新版本：

```
pip install --upgrade pyinstaller
```

或者安装开发者版本：

```
pip install https://github.com/pyinstaller/pyinstaller/archive/develop.tar.gz
```

当然了，也可以下载`whl`文件，然后`pip install`安装

更多可参考官网指引：

> http://www.pyinstaller.org/downloads.html

## 2. 打包初体验

我们简单试下打包python代码为exe可执行文件，测试代码如下：

```
# 测试.py
import os

path = os.getcwd()
print(f'当前文件路径：{path}')
os.system('pause')
```

os.system("pause");这种方法需要包含os模块（import os），在windows下IDLE运行会弹出cmd命令行，进行暂停操作，直接运行.py文件会直接在命令行中暂停。

这段代码是打印文件所在的目录，我们用`pyinstaller`简单打包的命令如下：

```
pyinstaller -F 测试.py
```

这个命令，执行过程如下：

```
(env_test) F:\PythonCool\pyinstaller>pyinstaller -F 测试.py    
403 INFO: PyInstaller: 4.3
403 INFO: Python: 3.8.10 (conda)
434 INFO: Platform: Windows-10-10.0.19042-SP0
436 INFO: wrote F:\PythonCool\pyinstaller\测试.spec
455 INFO: UPX is not available.
468 INFO: Extending PYTHONPATH with paths
['F:\\PythonCool\\pyinstaller', 'F:\\PythonCool\\pyinstaller']
501 INFO: checking Analysis
...
...
15006 INFO: Appending archive to EXE F:\PythonCool\pyinstaller\dist\测试.exe
18999 INFO: Building EXE from EXE-00.toc completed successfully.
```

成功后会在同级目录下生成一个`dist`文件夹，里面就是一个和代码文件名同名的可执行文件：

![图片](https://mmbiz.qpic.cn/mmbiz_png/vQr6oPKZqgsMkZwOhhr417iaBrJpHLhjMyokCKf61YUGKzh9FffwKuWW3PzTzNc0ShQRuPN6zce0zPW23QWnoAg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

双击该可执行文件，我们可以看到直接在`python`解释器里运行`测试.py`文件时一样的结果：

![图片](https://mmbiz.qpic.cn/mmbiz_png/vQr6oPKZqgsMkZwOhhr417iaBrJpHLhjMxLCbnHzKdQssXiaNfONIEPOVm5syrVTIIRaDWJyjFz12KxfwFSahiaiaA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

这里需要注意的是，我们在进行打包的时候，有必要指定被打包的py文件的路径，两种方式供选择：

方式一：**先切换到被打包py文件目录，再执行打包指令**

```
(base) C:\Users\Gdc>cd F:\PythonCool\pyinstaller
(base) C:\Users\Gdc>F:
(base) F:\PythonCool\pyinstaller>pyinstaller -F 测试.py
```

方式二：**打包指令中指定py文件的绝对路径**

```
(base) C:\Users\Gdc>pyinstaller -F F:\PythonCool\pyinstaller\测试.py
```

关于成功打包的`测试.exe`可执行文件，我们发现其图标是默认的，且启动时会显示命令行窗口。那么，我们可以怎么自定义`exe`图标，又或者去掉命令行窗口呢？

## 3. 打包进阶体验

好了，接下来，我们先看看关于`pyinstaller`打包时候的一些别的参数都有哪些，如何自定义`exe`图标以及如何去掉命令行窗口等等。

```
(env_test) F:\PythonCool\pyinstaller>pyinstaller -h
```

`pyinstaller -h`可以查看其参数说明，由于较多这里不做完整展示，摘取部分常用参数做简要介绍：

| 参数 | 说明                                                         |
| :--: | :----------------------------------------------------------- |
|  -F  | 产生单个的可执行文件                                         |
|  -D  | 产生一个目录（包含多个文件）作为可执行程序                   |
|  -a  | 不包含 Unicode 字符集支持                                    |
|  -d  | debug 版本的可执行文件                                       |
|  -w  | 指定程序运行时不显示命令行窗口（仅对 Windows 有效）          |
|  -c  | 指定使用命令行窗口运行程序（仅对 Windows 有效）              |
|  -o  | 指定 spec 文件的生成目录。如果没有指定，则默认使用当前目录来生成 spec 文件 |
|  -p  | 设置 Python 导入模块的路径（和设置 PYTHONPATH 环境变量的作用相似）。也可使用路径分隔符（Windows  使用分号，Linux 使用冒号）来分隔多个路径 |
|  -n  | 指定项目（产生的 spec）名字。如果省略该选项，那么第一个脚本的主文件名将作为 spec 的名字 |

**打包一个带自定义icon的exe可执行文件**

我们可以去这里下载icon文件：

> https://www.iconfont.cn/

可以去这里将图片转化为icon文件：

> https://www.bitbug.net/

然后，用一下命令可以自定义exe图标：

```
(env_test) F:\PythonCool\pyinstaller>pyinstaller -F -i icon.ico 测试.py 
```

成功后，我们可以看到图标变成了我们自定义的这个：

![图片](https://mmbiz.qpic.cn/mmbiz_png/vQr6oPKZqgsMkZwOhhr417iaBrJpHLhjMtpChNmX3oyh7e9YlYS1ia7FCNrFVxFQd2nK3WiamMqLiaFeSPloHGbRdw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**打包去掉命令行弹窗的exe可执行文件**

如果我们是有GUI的程序，想在启动的时候去掉命令行窗口，那么可以用以下指令进行打包，这里以`tkinter`内置GUI库为例展示：

```
# 测试.py
import tkinter

top = tkinter.Tk()
# 进入消息循环
top.mainloop()
```

以上测试代码，如果用初体验中的方式，在GUI界面出现的同时也会出现命令行弹窗，我们想去掉命令行弹窗可以：

```
(env_test) F:\PythonCool\pyinstaller>pyinstaller -F -w -i icon.ico 测试.py  
```

双击打包后的`exe`文件，可以看到只会出现GUI界面，命令行窗口并没有出现。

![图片](https://mmbiz.qpic.cn/mmbiz_png/vQr6oPKZqgsMkZwOhhr417iaBrJpHLhjMGZqvX703FV6FIbTpWNuDqw6NIhyib4krlFVibOZDPdhxywHOc2T36ibSA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## 4. 带配置文件打包

所谓带配置文件打包，这里是指打包的时候除了py文件、依赖的库之外，还存在需要引用的其他资源文件。直接用以上方式打包的时候，这些资源是无法被打进包的，我们需要进行修改打包时的`spec`文件来实现。

spec文件是告诉Pyinstaller怎么打包py文件，比如路径、资源、动态库、隐式调用的模块等等。一般来说，我们不需要对它进行修改...

这里我用此前《[词云绘制小工具](http://mp.weixin.qq.com/s?__biz=MzUzODA5MjM5NQ==&mid=2247489183&idx=1&sn=5cb0fdcdca821f26103591837ff650e3&chksm=fadda09bcdaa298d49f78391f229fe50ba442f93a5be5e5f09a1e0ab88bf04f619a9248898c1&scene=21#wechat_redirect)》的案例来进行介绍。

我们直接用打包进阶体验中的命令可以进行成功打包，不过这里我们发现有两个问题：①包体很大，比此前案例里大了10倍左右；②启动exe文件的时候报错了。

![图片](https://mmbiz.qpic.cn/mmbiz_png/vQr6oPKZqgsMkZwOhhr417iaBrJpHLhjMHG7uqeZVVeiabSbb0qgZ782YMZCzsQibDAoHdglFl3wv9tTZPKqQnmgQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

关于包体较大的情况，可以试着创建虚拟环境，然后只安装程序里需要调用的库即可，这里只简单介绍：

```
# 创建虚拟环境
conda create -n your_env_name python=3.8.10
# 启动虚拟环境
activate your_env_name
```

关于启动报错的情况，由于比较复杂，我们一步一步来看：

由于无命令行弹窗，无法查看到具体的报错，这里先去带命令行窗口形式看下报错信息，我们看报错如下：

![图片](https://mmbiz.qpic.cn/mmbiz_png/vQr6oPKZqgsMkZwOhhr417iaBrJpHLhjMMpQlw5WibXLCLZotH1bkyLDqV95NTE3bddKCia69kuforte9cRnslTfQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

提示缺少这个文件，我们可以在打包生成的`词云绘制工具.spec`配置文件里将这个资源放上

```
# -*- mode: python ; coding: utf-8 -*-
# 词云绘制工具.spec

block_cipher = None

a = Analysis(['词云绘制工具.py'],
             pathex=['F:\\PythonCool\\pyinstaller'],
             binaries=[],
             datas=[], # 这里带上资源文件地址
             hiddenimports=[], # 动态引入的库或模块
             hookspath=[],
             runtime_hooks=[],
             excludes=[],
             win_no_prefer_redirects=False,
             win_private_assemblies=False,
             cipher=block_cipher,
             noarchive=False)
pyz = PYZ(a.pure, a.zipped_data,
             cipher=block_cipher)
exe = EXE(pyz,
          a.scripts,
          a.binaries,
          a.zipfiles,
          a.datas,
          [],
          name='词云绘制工具',
          debug=False,
          bootloader_ignore_signals=False,
          strip=False,
          upx=True,
          upx_exclude=[],
          runtime_tmpdir=None,
          console=True , icon='icon.ico')
```

通过在`wordcloud`模块目录里查到了`stopwords`文件，我们将其放到data中。

![图片](https://mmbiz.qpic.cn/mmbiz_png/vQr6oPKZqgsMkZwOhhr417iaBrJpHLhjMRUf4OsF8x3RqqD36NOS6b5hPBibn0y4L8jHMUEibYWPaQf6UDKL94R9A/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

```
datas=[('C:\\Users\\Gdc\\anaconda3\\envs\\env_test\\Lib\\site-packages\\wordcloud\\stopwords','wordcloud')], # 这里带上资源文件地址
```

前者是资源文件在本机的位置，后者为打包后文件调用的相对路径，编辑好`spec`文件后，通过以下命令进行打包：

```
(env_test) F:\PythonCool\pyinstaller>pyinstaller -D 词云绘制工具.spec  
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/vQr6oPKZqgsMkZwOhhr417iaBrJpHLhjMyJfrT7l3YW59iayiabblpHDia6kYjqbFbYJDZQiau9gnQDa6UKobFOEVQA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

好吧，还有一些文件未被打进包，所以又出现同样的问题了。所以，我们是需要把全部的资源文件都加到spec文件里的data中。

我们找到全部的资源文件全部加上吧，然后再执行打包命令。

```
datas=[('C:\\Users\\Gdc\\anaconda3\\envs\\env_test\\Lib\\site-packages\\stylecloud\\static','stylecloud\\static'),
    ('C:\\Users\\Gdc\\anaconda3\\envs\\env_test\\Lib\\site-packages\\wordcloud\\stopwords','wordcloud'),
    ('C:\\Users\\Gdc\\anaconda3\\envs\\env_test\\Lib\\site-packages\\jieba\\analyse\\idf.txt','jieba\\analyse'),
    ('C:\\Users\\Gdc\\anaconda3\\envs\\env_test\\Lib\\site-packages\\jieba\\dict.txt','jieba')]
```

我们将配置资源打进包后可以正常启动exe可执行文件了。

![图片](https://mmbiz.qpic.cn/mmbiz_png/vQr6oPKZqgsMkZwOhhr417iaBrJpHLhjMWVib8snVtlVCqvySMByBgGuov9eFBaald69AK3JF0skWeDBz54EpEGA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

但是，又发现在执行词云绘制的时候，也会出现报错。不过看报错的情况是提示`不存在xx模块`，那么这是什么情况呢？！

## 5. 添加隐式调用库

我们找到报错的地方代码如下，采用了`__import__()`函数用于动态加载类和函数`palettable`模块。

```
def gen_palette(palette: str):
    """Generates the corresponding palette function from `palettable`."""
    palette_split = palette.split(".")
    palette_name = palette_split[-1]

    # https://stackoverflow.com/a/6677505
    palette_func = getattr(
        __import__(
            "palettable.{}".format(".".join(palette_split[:-1])),
            fromlist=[palette_name],
        ),
        palette_name,
    )
    return palette_func
```

对于这个问题，我试过两种方案，大家可以参考一下。

方案一：**在spec文件中hiddenimports中添加动态引用的模块**

```
hiddenimports=['palettable'], # 动态引入的库或模块
```

这种情况下，`palettable`库里也有一些配置文件需要添加到spec文件里的data中

```
('C:\\Users\\Gdc\\anaconda3\\envs\\env_test\\Lib\\site-packages\\palettable\\colorbrewer\\data','palettable\\colorbrewer\\data')
```

方案二：**修改stylecloud库中调用palettable模块的代码部分**

```
import palettable
def gen_palette(palette: str):
    palette_func = getattr(palettable.tableau,'BlueRed_6')
    return palette_func
 
    # """Generates the corresponding palette function from `palettable`."""
    # palette_split = palette.split(".")
    # palette_name = palette_split[-1]

    #    https://stackoverflow.com/a/6677505
    # palette_func = getattr(
        # __import__(
            # "palettable.{}".format(".".join(palette_split[:-1])),
            # fromlist=[palette_name],
        # ),
        # palette_name,
    # )
```

通过第4和5部分，我们用pyinstaller终于成功打包且正常运行使用了。

![图片](https://mmbiz.qpic.cn/mmbiz_png/vQr6oPKZqgsMkZwOhhr417iaBrJpHLhjMuKCOzqLKH68sOyibKj2ne6TPgVaYeVdrmjRbo6u9fu10icAjW8icIuuAw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

以上就是本次全部内容，大家如果遇到打包时涉及到配置文件的或者隐式调用的，可以采用这两个2技巧进行特殊打包！

不过，关于pyinstaller打包其实还有更多高级操作，大家可以多看看官方文档了解，主要是**命令行参数**及**spec文件里的配置要点**。