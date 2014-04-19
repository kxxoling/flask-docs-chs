.. _deferred-callbacks:

延迟请求回调
==========================

根据 Flask 的设计原则，响应对象被创建后可能会在一条回调链中传递，
对于这条回调链中的任意一个回调，你都可以修改或者将其替换掉。刚开始处理请求时，
响应对象尚未创建，但它必定会被被某个视图函数或者系统的其他组件按照需要来创建。

但是，如果你想要在响应对象尚未创建的时候修改响应信息，这时会发生什么呢？
一个常见的例子是，某个 before-request 函数想要在响应对象上设定 Cookie 。

一种解决方法是修改代码逻辑以回避这一情形，将这一部分代码迁移到
after-request 回调中。然而有些时候，这种迁移并不是一个容易，
而且可能使代码看起来非常糟糕。

另一可行的替代方法是，将这些回调函数绑定到 :data:`~flask.g` 对象中，然后在
请求结束的时候调用他们。通过这种方法，你可以在应用的任何一个地方来指定
代码延迟执行。


装饰器
-------------

下面这个装饰器就是关键所在，它将一个函数注册到 :data:`~flask.g` 对象上的
回调列表中::

    from flask import g

    def after_this_request(f):
        if not hasattr(g, 'after_request_callbacks'):
            g.after_request_callbacks = []
        g.after_request_callbacks.append(f)
        return f


调用延迟函数
--------------------

现在你可以使用 `after_this_request` 装饰器将一个函数标记为请求结束后执行，
但是我们仍然需要手动调用它们。为此，该函数仍需注册在 
:meth:`~flask.Flask.after_request` 回调中 ::

    @app.after_request
    def call_after_request_callbacks(response):
        for callback in getattr(g, 'after_request_callbacks', ()):
            response = callback(response)
        return response


一个实际案例
-------------------

现在我们可以随时将一个函数注册为在特定请求结束后执行。例如，你可以
在 before-request 中将用户当前语言的信息保存在 Cookie 中::

    from flask import request

    @app.before_request
    def detect_user_language():
        language = request.cookies.get('user_lang')
        if language is None:
            language = guess_language_from_request()
            @after_this_request
            def remember_language(response):
                response.set_cookie('user_lang', language)
        g.language = language
