.. _app-factories:

应用程序的工厂函数
==================

如果你已经开始在应用中使用包和 Blueprint（ :ref:`blueprints` ）来辅助开发，
这里还有一些非常好的办法可以进一步提升开发体验。在导入 blueprint 时创建应用对象就是一个常见模式。
如果你将应用对象的创建工作封装为一个函数来完成，那么就可以在此后创建它的多个实例。

为什么需要这样做呢？

1.  测试。你可以使用多个应用程序的实例，为每个实例分配分配不同的配置，
    从而测试每一种不同的情况。
2.  多个实例。比如你需要同时运行同一个应用的不同版本。当然，你可以
    在 web 服务器中设置配置不同的多个实例，但是如果你使用工厂函数，
    就可以随手在同一个进程中运行这应用的不同实例。

那么该如何使用呢？


基础的工厂函数
--------------

核心思想在于从一个函数里创建应用，像这样::

    def create_app(config_filename):
        app = Flask(__name__)
        app.config.from_pyfile(config_filename)

        from yourapplication.views.admin import admin
        from yourapplication.views.frontend import frontend
        app.register_blueprint(admin)
        app.register_blueprint(frontend)

        return app

有得必有失，导入时你无法在蓝图中使用这个应用程序对象，不过可以在请求中使用它。
如果获取当前配置下的对应的应用程序对象呢？请使用:
:data:`~flask.current_app` 函数::

    from flask import current_app, Blueprint, render_template
    admin = Blueprint('admin', __name__, url_prefix='/admin')

    @admin.route('/')
    def index():
        return render_template(current_app.config['INDEX_TEMPLATE'])

在这里我们从配置中查找网页模板文件的名字。


工厂 & 扩展
-----------

建议自己实现应用扩展和工厂函数，这样扩展对象不会和应用耦合地太过紧密。

以 `Flask-SQLAlchemy <http://pythonhosted.org/Flask-SQLAlchemy/>`_ 为例，
你不应该这样写： ::

    def create_app(config_filename):
        app = Flask(__name__)
        app.config.from_pyfile(config_filename)

        db = SQLAlchemy(app)

而应该在 model.py （或者其它等价物）中： ::

    db = SQLAlchemy()

然后在 application.py （或者其它等价物）中： ::

    def create_app(config_filename):
        app = Flask(__name__)
        app.config.from_pyfile(config_filename)

        from yourapplication.model import db
        db.init_app(app)

借助本设计模式，可以将应用相关的状态剥离到扩展中，这样扩展就可以应用于多个应用。
至于如何设计扩展请参阅 :doc:`/extensiondev` 。


使用应用
--------

因此，使用应用前首先需要创建应用对象，这是一个 `run.py` 示例文件::

    from yourapplication import create_app
    app = create_app('/path/to/config.cfg')
    app.run()


改进工厂函数
------------

前文所提供的工厂函数并不够智能，还可以继续改进，下面是些简单实用的修改建议：

1. 允许在单元测试中传入配置，这样可以避免在文件系统中创建多个配置文件。
2. 应用初始化时从 blueprint 中调用一个函数，这样就可以修改应用属性。
   （类似于请求前/后处理器中的 hook）
3. 如果有必要，在创建应用时引入 WSGI 中间件。
