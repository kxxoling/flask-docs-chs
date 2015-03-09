.. _app-context:

应用上下文
=======================

.. versionadded:: 0.9

Flask 背后的设计理念之一就是，代码在执行时会处于两种不同的“状态”（states）。
当 :class:`Flask` 对象被实例化后，在模块层次上便处于应用配置状态,
它将会在第一个请求到达时隐式地结束。当应用处于该状态的时候，
你可以认为下面的假设是成立的：

-   程序员可以安全地修改应用对象
-   目前还没有处理任何请求
-   必须通过应用对象的引用来修改它，不会有某个神奇的代理能够指向
    你刚刚创建或者修改的应用对象

相反，到了第二个状态，在处理请求时，有一些其它的规则:

-   当请求激活时，局部上下文对象（ :data:`flask.request` 等）
    指向当前的请求
-   你可以在任何时间使用代码与这些对象通信


这里还有与上面不同的第三种情况，有时你可能需要在没有活动请求的情况下，使用类似请求处理的方式
与应用交互，比如在交互式的 Python shell 中与应用，或者命令行应用交互的情况。

:data:`~flask.current_app` current_app 上下文局部环境就是由应用上下文驱动的。

应用上下文的作用
-----------------

应用上下文存在的主要原因在于，过去请求上下文被附加了一堆功能，但是又没
有更好的解决方案，而Flask 设计的支柱之一在于，允许一个 Python 进程中
拥有多个应用。

那么代码如何找到“正确的”应用？在过去，我们推荐显式地传递应用，但是这
会让我们在使用其它设计理念的库时遇到问题。

常用解决方法是使用后面将会提到的 :data:`~flask.current_app` 代理对象，
它与当前请求的应用的引用相绑定。然而，在没有请求时创建一个这样的请求上下文
是一个没有必要的昂贵操作，于是我们引入了应用上下文。

创建应用上下文
-----------------

创建应用上下文的方式有两种。第一种是隐式的：当请求上下文被传入时，
只要有必要就会创建一个应用上下文。因此，如果不需要对其进行操作，可以忽略它。

第二种是显式地调用 :meth:`~flask.Flask.app_context` 方法::

    from flask import Flask, current_app

    app = Flask(__name__)
    with app.app_context():
        # within this block, current_app points to app.
        print current_app.name

只要配置了 ``SERVER_NAME`` ，应用上下文还将用于 :func:`~flask.url_for` 函数，
这使得可以在没有请求的情况下生成 URL 。


应用上下文对象的区域性
-----------------------

应用上下文可以根据需要创建或者销毁，它不会在线程间移动，并且被不同的请求共享。
正因为如此，它非常适合存储数据库连接信息等。
内部栈对象叫做 :data:`flask._app_ctx_stack` 。扩展可以在最顶层自由地存储额外信息，
只要不出现重名问题，但不能是在 :data:`flask.g` 对象里，:data:`flask.g` 是为用户代码准备的。

更多内容见 :ref:`extension-dev` 。

上下文用法
----------

上下文的一个典型应用场景在于缓存一些我们需要在请求或者用例中使用的资源，
比如数据库连接。当我们在应用上下文中存储东西时你
得选择一个唯一的名字，因为应用上下文为 Flask 应用和扩展所共享。

最常见的应用就是把资源的管理分成如下两个部分：

1.  一个缓存在上下文中的隐式资源
2.  当上下文被销毁时重新分配基础资源

通常说来，如果 ``X`` 尚不存在，将会调用 ``get_X()`` 函数来创建资源，如果存在则直接返回它。
此外，还有一个名为 ``teardown_X()`` 的回调函数用于销毁资源 ``X`` 。

下面是我们刚刚提到的连接数据库的例子：::

    import sqlite3
    from flask import g

    def get_db():
        db = getattr(g, '_database', None)
        if db is None:
            db = g._database = connect_to_database()
        return db

    @app.teardown_appcontext
    def teardown_db(exception):
        db = getattr(g, '_database', None)
        if db is not None:
            db.close()

当 ``get_db()`` 这个函数第一次被调用的时候数据库连接已经被建立了。
为了看起来更隐式一点，我们可以使用 :class:`~werkzeug.local.LocalProxy` 这个类：::

    from werkzeug.local import LocalProxy
    db = LocalProxy(get_db)

这样用户就可以直接通过访问 ``db`` 在内部完成对 ``get_db()`` 的调用。
