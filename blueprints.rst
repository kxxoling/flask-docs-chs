.. _blueprints:

用蓝图实现模块化的应用
====================================

.. versionadded:: 0.7

Flask 使用 *Blueprint* 这一概念制作组件，或者实现应用内或应用间通用模式共享。
Blueprint 能够大幅简化大型应用的工作，并为 Flask 扩展提供了在应用上注册操作的核心方法。
:class:`Blueprint` 对象与 :class:`Flask` 应用对象的工作方式很像，
但它并不是一个应用，而是构建或扩展应用的*规划*。

为什么使用 Blueprint？
--------------------

Flask 中的 Blueprint 为这些情况设计:

* 把一个应用分解为 Blueprint 的集合。这理想的大型应用的组织方式，项目启动时，
  首先实例化一个应用对象、初始化扩展，再注册一系列的蓝图。
* 根据 URL 前缀或子域名在应用上注册 Blueprint。 URL 前缀/子域名中的参数即
  成为该 Blueprint 所有视图函数的参数（默认情况下）。
* 在一个应用中用不同的 URL 规则多次注册一个蓝图。
* 通过 Blueprint 提供模板过滤器、静态文件、模板和其它功能。Blueprint 并非必须实现应
  用或者视图函数。
* 出于以上情况，在 Flask 扩展初始化时注册 Blueprint。

Flask 中的 Blueprint 并非即插应用的，因为它实际上并不是一个应用——它是可以注册到应用上，
甚至可以多次注册的操作集合。为什么不使用多个应用对象？当然可以（见 :ref:`app-dispatch` ），
但是这些应用的配置将会因此分开，在 WSGI 层统一管理。

而 Blueprint 在 Flask 层提供了隔离，可以共享应用配置，并且在必要情况下可以更改所
注册的应用对象。
它的缺点在于，应用创建后不能撤销 Blueprint 的注册，只能销毁整个应用对象。

Blueprint 的理念
-------------------------

The basic concept of blueprints is that they record operations to execute when registered on an application. Flask associates view functions with blueprints when dispatching requests and generating URLs from one endpoint to another.


我的第一个 Blueprint
------------------

最基本的 Blueprint 大概就是这个样子。在这个示例中，我们要实现的是一个简单的、
渲染静态模板的蓝图::

    from flask import Blueprint, render_template, abort
    from jinja2 import TemplateNotFound

    simple_page = Blueprint('simple_page', __name__,
                            template_folder='templates')

    @simple_page.route('/', defaults={'page': 'index'})
    @simple_page.route('/<page>')
    def show(page):
        try:
            return render_template('pages/%s.html' % page)
        except TemplateNotFound:
            abort(404)

当我们使用 ``@simple_page.route`` 装饰器绑定函数时，在蓝图之后被注册时它
会记录把 `show` 函数注册到应用上的意图。此外，它会给函数的端点加上
由 :class:`Blueprint` 的构造函数中给出的蓝图的名称作为前缀（在此例
中是 ``simple_page`` ）。

注册 Blueprint
--------------

如何注册 Blueprint？像这样::

    from flask import Flask
    from yourapplication.simple_page import simple_page

    app = Flask(__name__)
    app.register_blueprint(simple_page)

这时检查已注册到应用的规则，可以看到这些::

    [<Rule '/static/<filename>' (HEAD, OPTIONS, GET) -> static>,
     <Rule '/<page>' (HEAD, OPTIONS, GET) -> simple_page.show>,
     <Rule '/' (HEAD, OPTIONS, GET) -> simple_page.show>]

第一个显然是来自应用自身，用于静态文件。其它的两个用于 ``simple_page``
蓝图中的 `show` 函数。如你所见，它们的前缀是 Blueprint 的名称，使用点
（ ``.`` ）来分割。

不过，Blueprint 还可以挂载在不同的位置::

    app.register_blueprint(simple_page, url_prefix='/pages')

生成出的规则会是这样::

    [<Rule '/static/<filename>' (HEAD, OPTIONS, GET) -> static>,
     <Rule '/pages/<page>' (HEAD, OPTIONS, GET) -> simple_page.show>,
     <Rule '/pages/' (HEAD, OPTIONS, GET) -> simple_page.show>]

在此之上，你可以多次注册 Blueprint，虽然它们并不都能正确地响应。实际上，
Blueprint 能否被多次挂载，取决于它是怎样实现的。


Blueprint 资源
-------------------

Blueprint 也可以提供资源。有时候你会只为它提供的资源而引入一个蓝图。

Blueprint 资源文件夹
`````````````````````````

像常规的应用一样，Blueprint 被设想为包含在一个文件夹中。当多个蓝图源于同一个文件
夹时，可以不必考虑上述情况，但也这通常不是推荐的做法。

这个文件夹会从 :class:`Blueprint` 的第二个参数中推断出来，通常是 `__name__` 。
这个参数决定对应蓝图的是哪个逻辑的 Python 模块或包。如果它指向一个存在的
Python 包，这个包（通常是文件系统中的文件夹）就是资源文件夹。如果是一个模块，
模块所在的包就是资源文件夹。你可以访问 :attr:`Blueprint.root_path` 属性来查看
资源文件夹是什么::

    >>> simple_page.root_path
    '/Users/username/TestProject/yourapplication'

可以使用 :meth:`~Blueprint.open_resource` 函数来快速从这个文件夹打开源文件::

    with simple_page.open_resource('static/style.css') as f:
        code = f.read()

静态文件
````````````

一个蓝图可以通过 `static_folder` 关键字参数提供一个指向文件系统上文件夹的路
径，来暴露一个带有静态文件的文件夹。这可以是一个绝对路径，也可以是相对于蓝图
文件夹的路径::

    admin = Blueprint('admin', __name__, static_folder='static')

默认情况下，路径最右边的部分就是它在 web 上所暴露的地址。因为这里这个文件夹
叫做 ``static`` ，它会在 蓝图 + ``/static`` 的位置上可用。也就是说，蓝图为
``/admin`` 把静态文件夹注册到 ``/admin/static`` 。

最后是命名的 `blueprint_name.static` ，这样你可以生成它的 URL ，就像你对应用
的静态文件夹所做的那样::

    url_for('admin.static', filename='style.css')

模板
`````````
如果你想要蓝图暴露模板，你可以提供 :class:`Blueprint` 构造函数中的
`template_folder` 参数来实现::

    admin = Blueprint('admin', __name__, template_folder='templates')

像对待静态文件一样，路径可以是绝对的或是相对蓝图资源文件夹的。模板文件夹会
被加入到模板的搜索路径中，但是比实际的应用模板文件夹优先级低。这样，你可以
容易地在实际的应用中覆盖蓝图提供的模板。

那么当你有一个 ``yourapplication/admin`` 文件夹中的蓝图并且你想要渲染
``'admin/index.html'`` 模板，且你已经提供了 ``templates`` 作为
`template_folder` ，你需要这样创建文件:
``yourapplication/admin/templates/admin/index.html``

构造 URL
-------------

当你想要从一个页面链接到另一个页面，可以像通常一个样使用 :func:`url_for`
函数，只需要在 URL 的 endpoint 前加上 Blueprint 的名称和一个点（ ``.`` ）作为前缀::

    url_for('admin.index')

如果你想要在 Blueprint 的视图函数或是模板中链接到同一 Blueprint 下另一个 endpoint，
可以仅仅在 endpoint 前加上一个点作为前缀::

    url_for('.index')

这个例子中，假设请求被分派到了 admin Blueprint endpoint，它实际上会链接到 ``admin.index`` 。