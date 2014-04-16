自定义错误页面
==================

Flask 自带了很顺手的 :func:`~flask.abort` 函数，通过在前面添加 HTTP 失败代码
中断请求，它还提供一个朴素的错误页面，包含一些基础的描述，并没有绚丽的设计。

根据错误代码的不同，用户看到某个错误的可能性大小也不同。

常见的错误代码
------------------

下面列出了一些用户经常遇到的错误代码，即使在这个应用毫无错误的情况下也可能发生:

*404 Not Found*
    经典的“哎呦，您输入的 URL 有误”消息。这个消息太常见了，即使是
    互联网的新手也知道 404 代号的意义: 该死，我寻找的东西不在那儿。确保
    404 页面上有一些有用的信息是一个好主意，至少应该提供一个返回主页的链接。

*403 Forbidden*
    如果你的网站包含一些访问控制类型，你必须向非法的请求返回 403 错误代号。
    请确保用户不会在试图访问了一个禁止访问的资源后不知所措。

*410 Gone*
    你知道 404 Not Found 代号还有一个兄弟名为 410 Gone 么? 很少有人真正实现它，
    你可以考虑将其返回给对以前曾经存在、但是现在已经删除的资源的请求，而
    不是直接返回 404 。如果你还没有从数据库里永久删除这个文档，仅仅是将他们
    标记为删除，那么可以向用户展示这个消息，说明他们寻找的东西已经永远删除了。

*500 Internal Server Error*
    通常编程错误或者服务器过载时会返回这个错误代号。在这里放一个
    漂亮的页面是一个非常好的主意。因为您的应用 *总有一天* 会出现错误(请参考
    :ref:`application-errors` )


错误处理器
--------------

错误处理器是一个类似于视图函数的函数，但是它仅错误发生时被执行，
错误被当成一个参数传递进来。一般来说错误可能是 :exc:`~werkzeug.exception.HTTPException` ，
但是在有些情况下会是其他错误: 内部服务器的错误的处理器在被执行时，将会
同时得到被捕捉到的实际代码错误作为参数。

错误处理器使用 :meth:`~flask.Flask.errorhandler` 装饰器注册，并以错误代码为参数。
请记住 Flask *不会* 替您设置错误代码，所以请确保在返回 response 对象时，提供了
对应的 HTTP 状态代码。

如下实现了一个 “404 Page Not Found” 错误处理的例子::

    from flask import render_template

    @app.errorhandler(404)
    def page_not_found(e):
        return render_template('404.html'), 404

模板示例如下:

.. sourcecode:: html+jinja

   {% extends "layout.html" %}
   {% block title %}Page Not Found{% endblock %}
   {% block body %}
     <h1>Page Not Found</h1>
     <p>What you were looking for is just not there.
     <p><a href="{{ url_for('index') }}">go somewhere nice</a>
   {% endblock %}
