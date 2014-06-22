.. mongokit-pattern:

在 Flask 中使用 MongoKit
=========================

近些日子，使用基于文档的数据库而非 DBMS 变得越来越流行。
本模式将展示如何使用文档映射库 MongoKit 与 MongoDB 交互。

本模式需要一个可用的 MongoDB 服务器，并且已安装 MongoKit 库。

使用 MongoKit 有两种常用的方法，我们将会逐一介绍:

显式调用
-----------

MongoKit 的默认行为是这种显式调用的方法。这种方法跟 Django 或者 SQLAlchemy
扩展显示调用扩展想法大体上是一样的。

下面是一个 `app.py` 模块实例::

    from flask import Flask
    from mongokit import Connection, Document

    # 配置
    MONGODB_HOST = 'localhost'
    MONGODB_PORT = 27017

    # 创建应用对象
    app = Flask(__name__)
    app.config.from_object(__name__)

    # 连接数据库
    connection = Connection(app.config['MONGODB_HOST'],
                            app.config['MONGODB_PORT'])


要定义您的模型，只需编写一个从 MongoKit 导入的 `Document` 类的子类。如果您
已经看过了 SQLAlchemy 的方案，可能会奇怪为什么这里没有会话（session），甚至没有
定义 `init_db` 函数。一方面， MongoKit 并没有类似于会话的东西。这虽然可能会增加
代码量，但是同时也使得数据库操作非常高效。另一方面， MongoDB 是无模式的。
这意味着你在相同的插入查询，可以使用不同的数据结构。 MongoKit 本身也是无模式的，
但是实现了一些验证机制以确保数据完整性。

以下是一个例子 （你可以将这个也放进 `app.py` 文件里)::

    def max_length(length):
        def validate(value):
            if len(value) <= length:
                return True
            raise Exception('%s must be at most %s characters long' % length)
        return validate

    class User(Document):
        structure = {
            'name': unicode,
            'email': unicode,
        }
        validators = {
            'name': max_length(50),
            'email': max_length(120)
        }
        use_dot_notation = True
        def __repr__(self):
            return '<User %r>' % (self.name)

    # 使用当前连接注册 User 文档
    connection.register([User])


这个例子展示了如何定于自己的模式（命名结构）、一个最大字符长度
的验证器，并且使用了 Monkit 的一项名为 `use_dot_notation` 的特性。默认情况下
MongoKit 的使用类似于 Python 字典，但是将 `use_dot_notation` 设为 True 之后，你可以
像在其它 ORM 中那样，使用点运算符来分割属性的方式访问您的文档。

向数据库里添加数据的方法如下所示:

>>> from yourapplication.database import connection
>>> from yourapplication.models import User
>>> collection = connection['test'].users
>>> user = collection.User()
>>> user['name'] = u'admin'
>>> user['email'] = u'admin@localhost'
>>> user.save()

注意，MongoKit 对于列的类型要求有些严格，你必须使用 `unicode` 
作为 `name` 和 `email` 的类型，而不能是普通的 `str` 类型。

查询也很简单:

>>> list(collection.User.find())
[<User u'admin'>]
>>> collection.User.find_one({'name': u'admin'})
<User u'admin'>

.. _MongoKit: http://bytebucket.org/namlook/mongokit/


PyMongo 兼容层
---------------------------

如果你想直接使用 PyMongo，也可以通过 MongoKit 实现，如果对应用的性能还有要求，
可以采用这种方法。注意，例子并没有展示配合 Flask 使用的具体方法，
如有需要请参考上面 MongoKit 的例子代码::

    from MongoKit import Connection

    connection = Connection()

插入数据可以使用 `insert` 方法。首先我们需要拥有一个连接，这跟在 SQL 世界中使用表有些类似。

>>> collection = connection['test'].users
>>> user = {'name': u'admin', 'email': u'admin@localhost'}
>>> collection.insert(user)

print list(collection.find())
print collection.find_one({'name': u'admin'})

MongoKit 将会为我们自动提交修改。

查询数据库需要要直接使用数据库连接:

>>> list(collection.find())
[{u'_id': ObjectId('4c271729e13823182f000000'), u'name': u'admin', u'email': u'admin@localhost'}]
>>> collection.find_one({'name': u'admin'})
{u'_id': ObjectId('4c271729e13823182f000000'), u'name': u'admin', u'email': u'admin@localhost'}

返回的结果也同样是类似于字典的对象:

>>> r = collection.find_one({'name': u'admin'})
>>> r['email']
u'admin@localhost'

关于 MongoKit 的更多信息，请访问
`website <https://github.com/namlook/mongokit>`_.
