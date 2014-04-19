视图装饰器
===============

Python 有一个非常有趣的特性，叫做函数装饰器。该特性允许你的 Web 应用代码看起来十分工整。
Flask 中的视图都是一个函数装饰器，用来向函数中注入额外的功能。
你可能已经使用过了 :meth:`~flask.Flask.route` 装饰器，但在某些情况下你需要实现自己的装饰器。
例如，你有一个仅供登录后的用户访问的视图，如果未登录的用户试图访问，则把他们
转接到登录界面。这个例子很好地说明了装饰器的用武之地。

过滤未登录用户的装饰器
-------------------

现在让我们实现一个这样的装饰器。装饰器是指返回函数的函数，它其实非常简单。
你只需要记住，实现一个这样的东西，会更新函数的 `__name__` 、 `__module__`
以及其它一些属性，这点经常被遗忘。但是并不需要你亲自动手，我们
有一个专门处理这些的用作装饰器的函数(:func:`functools.wraps` )。

在这个例子中，用户登陆页面的名字是 ``'login'``，当前已登录用户被保存在 `g.user` 当中，
如果没有用户登录， `g.user` 会是 `None`::

    from functools import wraps
    from flask import g, request, redirect, url_for

    def login_required(f):
        @wraps(f)
        def decorated_function(*args, **kwargs):
            if g.user is None:
                return redirect(url_for('login', next=request.url))
            return f(*args, **kwargs)
        return decorated_function

你又该如何使用这些装饰器呢？将它加为视图函数外最里层的装饰器。添加更多
装饰器时，一定要记住 :meth:`~flask.Flask.route` 装饰器永远在最外层::

    @app.route('/secret_page')
    @login_required
    def secret_page():
        pass

缓存装饰器
-----------------

假设你有一个运算量很大的函数，你希望能够将其生成的结果在一段时间内缓存起来，
装饰器将会非常合适。我们假定你已经参考 :ref:`caching-pattern`
中提到的内容配置好了缓存功能。

这里有一个用作例子的缓存函数，它根据一个指定的前缀(通常是一个格式化字符串)
和当前请求的路径生成一个缓存键。请注意我们创建了一个这样的函数: 它先创建
一个装饰器，然后用这个装饰器包装目标函数。听起来很复杂？不幸的是，这的确
有些难，但是代码看起来会非常直接明了。

被装饰器包装的函数将能做到如下几点:

1. 以当前请求和路径为基础生成缓存时使用的键。
2. 从缓存中取出对应键的值，如果缓存返回的不是空，我们就将它返回回去。
3. 如果缓存中没有这个键，那么将会执行最初的函数，并将其返回的值在指定时间内
   (默认5分钟内)缓存起来。

代码如下::

    from functools import wraps
    from flask import request

    def cached(timeout=5 * 60, key='view/%s'):
        def decorator(f):
            @wraps(f)
            def decorated_function(*args, **kwargs):
                cache_key = key % request.path
                rv = cache.get(cache_key)
                if rv is not None:
                    return rv
                rv = f(*args, **kwargs)
                cache.set(cache_key, rv, timeout=timeout)
                return rv
            return decorated_function
        return decorator

注意，这段代码假定有一个可用的 `cache` 对象示例。请参考 :ref:`caching-pattern` 
以获取更多信息。


模板装饰器
--------------------

TurboGears 的家伙们前一段时间发明了一种新的常用范式，那就是模板装饰器。
这个装饰器的关键在于，将想要传递给模板的值组织成字典的形式，然后从
视图函数中返回，这个模板将会被自动渲染。因此，下面的三个例子是等价的::

    @app.route('/')
    def index():
        return render_template('index.html', value=42)

    @app.route('/')
    @templated('index.html')
    def index():
        return dict(value=42)

    @app.route('/')
    @templated()
    def index():
        return dict(value=42)

正如你所看到的，如果没有模板名被指定，那么他会使用 URL 映射的最后一部分，
然后将点转换为反斜杠，最后添加上 ``'.html'`` 作为模板的名字。被装饰的函数返回字典
会被传递给模板渲染函数，如果返回的是 `None` ，那么则相当于一个空字典。
如果非字典类型的对象被返回，函数将照原样将那个对象再次返回。
这样你就可以使用重定向函数或者返回简单的字符串了。

这是那个装饰器的源代码::

    from functools import wraps
    from flask import request

    def templated(template=None):
        def decorator(f):
            @wraps(f)
            def decorated_function(*args, **kwargs):
                template_name = template
                if template_name is None:
                    template_name = request.endpoint \
                        .replace('.', '/') + '.html'
                ctx = f(*args, **kwargs)
                if ctx is None:
                    ctx = {}
                elif not isinstance(ctx, dict):
                    return ctx
                return render_template(template_name, **ctx)
            return decorated_function
        return decorator


终端装饰器
------------------

如果你希望使用 werkzeug 的路由系统以获得更高的灵活性，需要将终点（Endpoint）
像 :class:`~werkzeug.routing.Rule` 中定义的那样与视图函数映射起来，配合这个装饰器
使用。例如::

    from flask import Flask
    from werkzeug.routing import Rule

    app = Flask(__name__)                                                          
    app.url_map.add(Rule('/', endpoint='index'))                                   

    @app.endpoint('index')                                                         
    def my_index():                                                                
        return "Hello world"     
