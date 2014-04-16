.. _app-dispatch:

应用调度
=======================

应用调度指的是在 WSGI 层次合并运行多个 Flask 应用的过程。你可以将 Flask 与其它 WSGI 应用合并，
但对于更大的东西则不允许了。这也就是说，你甚至可以将 Django 和 Flask 应用运行在同一解释器下。
当然，这么做的效果依赖于这些应用的内部实现。

这与 :ref:`模块化 <larger-applications>` 的根本区别在于，此时运行的不
同的 Flask 应用之间是完全独立的，他们使用了不同的配置，在 WSGI 层进行调度。


如何使用此文档
--------------------------

下面的所有技巧和例子最终都将得到一个 ``application`` 对象，这个对象
可以在任何 WSGI 服务器上运行。在生产环境下，请参看 :ref:`deployment` 相关章节。
Werkzeug 内置了开发服务器，在开发时可以通过 
:func:`werkzeug.serving.run_simple` 函数使用::

    from werkzeug.serving import run_simple
    run_simple('localhost', 5000, application, use_reloader=True)

注意，:func:`run_simple <werkzeug.serving.run_simple>` 函数不是为生产
用途设计的，部署应用时建议使用 :ref:`更成熟的 WSGI 服务器 <deployment>` 。

为了使用交互式调试器，必须在应用和简易开发服务器两边都激活调试模式。
下面是一个带有调试功能的 “Hello World” 的例子::

    from flask import Flask
    from werkzeug.serving import run_simple

    app = Flask(__name__)
    app.debug = True

    @app.route('/')
    def hello_world():
        return 'Hello World!'

    if __name__ == '__main__':
        run_simple('localhost', 5000, app,
                   use_reloader=True, use_debugger=True, use_evalex=True)


合并应用
----------------------

如果你有一些完全独立的应用程序，并希望他们使用同一个 Python 解释器，背靠背地运行，
可以使用 :class:`werkzeug.wsgi.DispatcherMiddleware` 这个类。
这里的思想是，每个 Flask 应用对象都是一个有效的 WSGI 应用对象，通过调度中间层
把它们合并成一个规模更大的应用。这通过 URL 前缀来实现调度。

例如，您可以使主应用运行在 `/` 路径，而后台接口运行在 `/backend` 路径::

    from werkzeug.wsgi import DispatcherMiddleware
    from frontend_app import application as frontend
    from backend_app import application as backend

    application = DispatcherMiddleware(frontend, {
        '/backend':     backend
    })


根据子域名调度
---------------------

有时，你会希望使用对一个应用使用不同的配置，每个配置运行一个实例，从而存在多个实例。
假设应用对象是在函数中生成的，你可以调用这个函数并实例化一个实例，这相当容易。
为了保证你的应用支持在函数中创建新的对象，请先参考 :ref:`app-factories` 模式。

为不同的子域名创建不同的应用对象就是一个相当典型的例子。比如，你通过 web 服务器，
将所有子域名的请求都转发给应用，接下来，你根据子域名信息分别创建针对不同用户的实例。
你只需要通过服务器将所有设置好所有子域名的监听，接下来可以通过 WSGI 动态地创建应用。

实现此功能最佳的抽象层就是 WSGI 层。您可以编写您自己的 WSGI 程序来检查访问请求，
然后委托给 Flask 应用，如果该应用尚未存在，那么将创建一个，并保存下来。

    from threading import Lock

    class SubdomainDispatcher(object):

        def __init__(self, domain, create_app):
            self.domain = domain
            self.create_app = create_app
            self.lock = Lock()
            self.instances = {}

        def get_application(self, host):
            host = host.split(':')[0]
            assert host.endswith(self.domain), 'Configuration error'
            subdomain = host[:-len(self.domain)].rstrip('.')
            with self.lock:
                app = self.instances.get(subdomain)
                if app is None:
                    app = self.create_app(subdomain)
                    self.instances[subdomain] = app
                return app

        def __call__(self, environ, start_response):
            app = self.get_application(environ['HTTP_HOST'])
            return app(environ, start_response)


可以这样使用调度器::

    from myapplication import create_app, get_user_for_subdomain
    from werkzeug.exceptions import NotFound

    def make_app(subdomain):
        user = get_user_for_subdomain(subdomain)
        if user is None:
            # 如果该子域名没有对应的用户，我们仍然需要返回 WSGI 应用来处理请求。
            # 可以返回 NotFound() exception，然后应用会渲染一个 404 页面。
            # 之后你也可以选择将用户跳转回主页面。
            return NotFound()

        # 否则针对用户创建应用
        return create_app(user)

    application = SubdomainDispatcher('example.com', make_app)


根据路径调度
----------------

通过 URL 路径分发请求跟前面的方法很相似。只需要简单检查请求路径当中到第一个
斜杠之前的部分，而非确定子域名的 `HOST` 头信息即可 ::

    from threading import Lock
    from werkzeug.wsgi import pop_path_info, peek_path_info

    class PathDispatcher(object):

        def __init__(self, default_app, create_app):
            self.default_app = default_app
            self.create_app = create_app
            self.lock = Lock()
            self.instances = {}

        def get_application(self, prefix):
            with self.lock:
                app = self.instances.get(prefix)
                if app is None:
                    app = self.create_app(prefix)
                    if app is not None:
                        self.instances[prefix] = app
                return app

        def __call__(self, environ, start_response):
            app = self.get_application(peek_path_info(environ))
            if app is not None:
                pop_path_info(environ)
            else:
                app = self.default_app
            return app(environ, start_response)

该例子与之前根据子域名调度的区别是，如果创建应用对象的函数返回了 `None`,
那么请求就被降级回推到另一个应用当中::

    from myapplication import create_app, default_app, get_user_for_prefix

    def make_app(prefix):
        user = get_user_for_prefix(prefix)
        if user is not None:
            return create_app(user)

    application = PathDispatcher(default_app, make_app)
