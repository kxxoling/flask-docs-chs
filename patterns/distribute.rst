.. _distribute-deployment:

部署和分发
=========================

`distribute`_ 的前身是 ``setuptools`` ，（正如其名）通常用于分发Python 库和扩展程序的外部库。
它扩展了 Python 预装的 ``distutils`` 库，后者是一个基础的模块安装系统，
该系统也支持很多简化大型应用发布的复杂构造。

- **支持依赖关系管理**: 一个库可以声明自己依赖于哪些软件包，从而在安装该
  模块的时候，自动安装依赖的软件包。
- **注册软件包**: setuptools 将包注册到你的 Python 安装环境中。
  这样一个包中的代码就查询另一个包的信息。该系统最知名的特性就是对入口机制的支持，
  也就是说一个包可以声明一个“入口”，从而允许其它包通过这个“入口”对自己进行扩展。
- **安装管理**: Python 默认安装了 `easy_install` ，它可以用于安装其它库。
  你也可以使用 `pip` 这个可能早晚会代替 `easy_install` 的包管理器，
  它能够完成更多安装包之外的任务。

而对于 Flask 而言，所有可以在 cheessshop 上找到的软件包，都支持 distribute 
分发管理器，或者更古老的 setuptools 和 distutils 分发。

在这里，我们假定你的应用名为 `yourapplication.py` ，使用
:ref:`package <larger-applications>` 而非模块的结构来组织代码。分发带有标准模块的
代码不被 `distribute`_ 支持，所以我们不去管它。
如果尚未将应用转化为包，请参考前文 :ref:`larger-applications` 学习如何实现。

实用 distribute 进行部署只是应用部署的入门，还有很多更复杂并且更自动化的部署方案，
如想要将其完全自动化，可以阅读 :ref:`fabric-deployment` 这一章。

基本安装脚本
-----------

因为你成功运行了 Flask，所以你的系统上一定有 setuptools 或者 distribute。
如果没有也别担心。这里帮你准备了一个安装脚本：``distribute_setup.py`` 。
你只需要下载并用 Python 解释器运行它。

考虑这些操作可能会有风险，因此建议您参考 :ref:`你最好使用 virtualenv
<virtualenv>` 一文。

您的安装代码将总是保存在与您应用同目录下的 `setup.py` 文件中。为文件
指定这一名称只是为了方便，不过一般来说每一个人自然而然的在程序目录下
寻找这个文件，所以您最好别改变它。

同时，即使您在使用 `distribute` ，您也会导入一个名为 `setuptools` 的包。
`distribute` 完全向下兼容 `setuptools` ，所以我们也使用这个名字来导入它。

一个基本的 Flask 应用的 `setup.py` 文件看起来像如下这样::

    from setuptools import setup

    setup(
        name='Your Application',
        version='1.0',
        long_description=__doc__,
        packages=['yourapplication'],
        include_package_data=True,
        zip_safe=False,
        install_requires=['Flask']
    )

切记，您必须详细地列出子代码包，如果您想要 distribute 自动为您寻找这些包，
您可以使用 `find_packages` 函数::

    from setuptools import setup, find_packages

    setup(
        ...
        packages=find_packages()
    )

大多数 `setup` 函数当中的参数的意义从字面意思就能看出来，然而
`include_package_data` 和 `zip_safe` 可能不在此列。
`include_package_data` 告诉 distribute 自动查找一个 `MANIFEST.in` 文件。
解析此文件获得有效的包类型的数据，并安装所有这些包。我们使用这个特性来分发
Python 模块自带的静态文件和模板(参考 :ref:`distributing-resources`)。而 `zip_safe` 
标志可以被用来强制阻止 ZIP 安装包的建立。通常情况下，您不希望您的包以 ZIP 压缩
包的形式被安装，因为一些工具不支持这种方式，而且这样也会让调试代码异常麻烦。

.. _distributing-resources:

分发代码
----------------------

如果您视图安装您刚刚创建的包，您会发现诸如 `static` 和 `templates` 这样的
文件夹没有安装进去。这是因为 distribute 不知道该把哪些文件添加进去。您只要
在 `setup.py` 相同的文件夹下创建一个 `MANIFEST.in` 文件，并在此文件中列出
所有应该被添加进去的文件::

    recursive-include yourapplication/templates *
    recursive-include yourapplication/static *

不要忘记，即使您已经将他们列在 `MANIFEST.in` 文件当中，也需要您将 `setup` 函数的
`include_package_data` 参数设置为 `True` ，否则他们仍然不会被安装。


声明依赖关系
----------------------

您需要使用一个链表在 `install_requires` 参数中声明依赖关系。链表的每个元素是
需要从 PyPI 下载并安装的包的名字，默认将总会下载安装最新的的版本。但是您也
可以指定需要的最大和最小的版本区间。以下是一个例子::

    install_requires=[
        'Flask>=0.2',
        'SQLAlchemy>=0.6',
        'BrokenPackage>=0.7,<=1.0'
    ]

前文曾经指出，这些依赖都从 PyPI 当中下载，如果您需要依赖一个不能在 PyPI 当中
被下载的包，比如这个包是个内部的，您不想与别人分享。这时，您可以依然照原来
那样将包列在列表里，但是同时提供一个包括所有可选下载地址的列表，以便于安装时
从这些地点寻找分发的软件包::

    dependency_links=['http://example.com/yourfiles']

请确认那个页面包含一个文件夹列表，且页面上的连接被指向实际需要下载的软件包。
distribute 通过扫描这个页面来寻找需要安装的文件，因此文件的名字必须是正确无误的。
如您有一个内部服务器包含有这些包，将 URL 指向这个服务器。


安装 / 开发
-----------------------

安装您的应用(到一个 virtualenv)，只需使用 `install` 指令运行 `setup.py` 即可。
这会将您的应用安装到一个 virtualenv 的 site-packages 文件夹下面，并且同时
下载和安装所有的依赖包::

    $ python setup.py install

如果您在进行基于这个包的开发，并且希望安装开发所依赖的工具或软件包，
您可以使用 `develop` 命令代替 `install` ::

    $ python setup.py develop

此时将不会把您的文件拷贝到 site-packages 文件夹，而仅仅是在那里创建指向
这些文件的文件链接。您可以继续编辑和修改这些代码，而无需在每次修改之后
运行 `install` 命令。


.. _distribute: http://pypi.python.org/pypi/distribute
.. _pip: http://pypi.python.org/pypi/pip
.. _distribute_setup.py: http://python-distribute.org/distribute_setup.py
