添加 Favicon
================

“Favicon” 是网页在浏览器标签页或者历史记录里显示的图标。
这个图标能帮助用户将你的网站与其他网站区分开，因此请使用一个独特的标志。

如何将一个 Favicon 添加到您的 Flask 应用中？首先，当然得先有一个可用的图标，
此图标应该是 16 x 16 像素的，且格式为 ICO 。这些虽然不是必需的规则，
但却是被所有浏览器所支持的事实标准。将这个图标放置到你的静态文件目录下，
命名名为 :file:`favicon.ico` 。

现在，为了让浏览器找到你的图标，正确的方法是添加一个 Link 标签到 HTML 当中。
如下:

.. sourcecode:: html+jinja

    <link rel="shortcut icon" href="{{ url_for('static', filename='favicon.ico') }}">

对于大多数浏览器来说这就足够了。然而一些非常古老的浏览器却不支持这个标准。
原来的标准是在网站的根路径下查找 favicon 文件并使用它。如果应用程序
不是挂在在域名的根路径，要么需要配置 Web 服务器来在根路径提供这一图标，
否则就无法实现这一功能了。然而，如果你的应用是在根路径，可以
简单地配置一条重定向的路由::

    app.add_url_rule('/favicon.ico',
                     redirect_to=url_for('static', filename='favicon.ico'))

如果想要保存额外的重定向请求，你也可以使用 :func:`~flask.send_from_directory` 
函数写一个视图函数::

    import os
    from flask import send_from_directory

    @app.route('/favicon.ico')
    def favicon():
        return send_from_directory(os.path.join(app.root_path, 'static'),
                                   'favicon.ico', mimetype='image/vnd.microsoft.icon')

我们可以不明确地指定 mimetype ，浏览器会自行猜测文件的类型。但是我们也可以
指定它以便于避免额外的猜测，因为这个 mimetype 总是固定的。

以上代码将会通过你的应用程序来提供图标文件的访问。然而，如果可能的话，
配置你的网页服务器来提供访问服务会更好。请参考对应网页服务器的文档。

参考
--------

* Wikipedia 上有关 `Favicon <http://en.wikipedia.org/wiki/Favicon>`_ 的文章
  
