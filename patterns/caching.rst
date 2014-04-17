.. _caching-pattern:

缓存
=======

如果你的应用运行得很慢，那么它应该引入一些缓存。好吧，至少这是提高其表现
最简单的方法。缓存的工作是什么？假设你的应用中存在一个比较费时的函数，
但是这个函数的返回结果可能在5分钟之内都足够有效，因此你可以将这个结果
放到缓存中一段时间以避免重复计算。

Flask 本身并不提供缓存功能，但是作为 Flask 基础的 Werkzeug 库则提供了一些
基础的缓存支持。Werkzeug 支持多种缓存后端，通常的选择是 Memcached 服务器。

配置缓存
------------------

和创建 :class:`~flask.Flask` 对象一样，你需要创建一个缓存对象，然后让它保持存在。
如果你使用的是开发服务器，可以创建一个 :class:`~werkzeug.contrib.cache.SimpleCache` 
对象，这个对象将元素缓存在 Python 解释器的内存中::

    from werkzeug.contrib.cache import SimpleCache
    cache = SimpleCache()

如果你希望使用 Memcached 进行缓存，请确保已经安装了 Memcache 模块支持
(可以通过 `PyPi<http://pypi.python.org/` 获取)，并且有一个可用的、正在运行的 
Memcached 服务器。然后你可以像下面这样连接到缓存服务器::

    from werkzeug.contrib.cache import MemcachedCache
    cache = MemcachedCache(['127.0.0.1:11211'])

如果你正在使用 App Engine ，可以通过下面的代码连接到 App Engine 的
缓存服务器::

    from werkzeug.contrib.cache import GAEMemcachedCache
    cache = GAEMemcachedCache()

使用缓存
-------------

关于缓存的使用有两个重要的函数。那就是 :meth:`~werkzeug.contrib.cache.BaseCache.get` 
函数和 :meth:`~werkzeug.contrib.cache.BaseCache.set` 函数。它们的使用方法如下:

从缓存中读取项目，使用 :meth:`~werkzeug.contrib.cache.BaseCache.get` 函数，
如果缓存中存在对应项目，将会返回其值；否则返回 `None` ::

    rv = cache.get('my-item')

向缓存中添加项目，使用 :meth:`~Werkzeug.contrib.cache.BaseCache.set` 函数。
第一个参数是想要设定的键，第二个参数是想要缓存的值。你还可以设定一个超时时间参数，
当时间超过时缓存系统将会自动清除这个项目。

以下是一个完整实现的例子::

    def get_my_item():
        rv = cache.get('my-item')
        if rv is None:
            rv = calculate_value()
            cache.set('my-item', rv, timeout=5 * 60)
        return rv
