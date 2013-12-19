---
layout: post
title: Flask-Admin入门
tags: flask web
category: it
---

{% raw %}

# Flask-Admin是什么？
Flask-Admin是Flask框架的一个扩展，用它能够快速创建Web管理界面，它实现了比如用户、文件的增删改查等常用的管理功能；如果对它的默认界面不喜欢，可以通过修改模板文件来定制；  

Flask-Admin把每一个菜单（超链接）看作一个view，注册后才能显示出来，view本身也有属性来控制其是否可见；因此，利用这个机制可以定制自己的模块化界面，比如让不同权限的用户登录后看到不一样的菜单；

# 如何使用
Flask-Admin的文档写得比较简单，单从文档很难理解如何使用，好在作者提供了的很多Example（见[源码](https://github.com/mrjoes/flask-admin)），基本上照葫芦画瓢即可；  

下面就通过分析一些Example代码来学习使用

# example/simple
这是最简单的一个样例，可以帮助我们快速、直观的了解基本概念，学会定制Flask-Admin的界面  

`simple.py`：

    from flask import Flask
    
    from flask.ext import admin

    
    # Create custom admin view
    class MyAdminView(admin.BaseView):
        @admin.expose('/')
        def index(self):
            return self.render('myadmin.html')

    
    class AnotherAdminView(admin.BaseView):
        @admin.expose('/')
        def index(self):
            return self.render('anotheradmin.html')
    
        @admin.expose('/test/')
        def test(self):
            return self.render('test.html')

    
    # Create flask app
    app = Flask(__name__, template_folder='templates')
    app.debug = True
    
    # Flask views
    @app.route('/')
    def index():
        return '<a href="/admin/">Click me to get to Admin!</a>'
    
    # Create admin interface
    admin = admin.Admin()
    admin.add_view(MyAdminView(category='Test'))
    admin.add_view(AnotherAdminView(category='Test'))
    admin.init_app(app)
    
    if __name__ == '__main__':
    
        # Start app
        app.run()

在[这里](http://examples.flask-admin.org/simple/admin/)可以看到运行效果  

##BaseView

所有的view都必须继承自`BaseView`：

    class BaseView(name=None, category=None, endpoint=None, url=None, static_folder=None, static_url_path=None)

* name: view在页面上表现为一个menu（超链接），menu name == 'name'，缺省就用小写的class name
* category: 如果多个view有相同的category就全部放到一个dropdown里面（dropdown name=='category'）
* endpoint: 假设endpoint='xxx'，则可以用`url_for(xxx.index)`，也能改变页面URL（`/admin/xxx`）
* url: 页面URL，优先级`url > endpoint > class name`
* static_folder: static目录的路径
* static_url_path: static目录的URL

`anotheradmin.html`:

    {% extends 'admin/master.html' %}
    {% block body %}
        Hello World from AnotherMyAdmin!<br/>
        <a href="{{ url_for('.test') }}">Click me to go to test view</a>
    {% endblock %}

如果`AnotherAdminView`增加参数`endpoint='xxx'`，那这里就可以写成`url_for('xxx.text')`，然后页面URL会由`/admin/anotheradminview/`变成`/admin/xxx`  

如果同时指定参数`url='aaa'`，那页面URL会变成`/admin/aaa`，url优先级比endpoint高


##Admin

    class Admin(app=None, name=None, url=None, subdomain=None, index_view=None, translations_path=None, endpoint=None, static_url_path=None, base_template=None)

* app: Flask Application Object；本例中可以不写`admin.init_app(app)`，直接用`admin = admin.Admin(app=app)`是一样的
* name: Application name，缺省'Admin'；会显示为main menu name('Home'左边的'Admin')和page title
* subdomain: ??? 
* index_view: 'Home'那个menu对应的就叫index view，缺省`AdminIndexView`
* base_template: 基础模板，缺省`admin/base.html`，该模板在Flask-Admin的源码目录里面

部分Admin代码如下：

    class MenuItem(object):
        """
            Simple menu tree hierarchy.
        """
        def __init__(self, name, view=None):
            self.name = name
            self._view = view
            self._children = []
            self._children_urls = set()
            self._cached_url = None
    
            self.url = None
            if view is not None:
                self.url = view.url
    
        def add_child(self, view):
            self._children.append(view)
            self._children_urls.add(view.url)

    class Admin(object):

        def __init__(self, app=None, name=None,
                     url=None, subdomain=None,
                     index_view=None,
                     translations_path=None,
                     endpoint=None,
                     static_url_path=None,
                     base_template=None):

            self.app = app
    
            self.translations_path = translations_path
    
            self._views = []
            self._menu = []
            self._menu_categories = dict()
            self._menu_links = []
    
            if name is None:
                name = 'Admin'
            self.name = name
    
            self.index_view = index_view or AdminIndexView(endpoint=endpoint, url=url)
            self.endpoint = endpoint or self.index_view.endpoint
            self.url = url or self.index_view.url
            self.static_url_path = static_url_path
            self.subdomain = subdomain
            self.base_template = base_template or 'admin/base.html'
    
            # Add predefined index view
            self.add_view(self.index_view)
    
            # Register with application
            if app is not None:
                self._init_extension()
    
        def add_view(self, view):

            # Add to views
            self._views.append(view)
    
            # If app was provided in constructor, register view with Flask app
            if self.app is not None:
                self.app.register_blueprint(view.create_blueprint(self))
                self._add_view_to_menu(view)
    
        def _add_view_to_menu(self, view):

            if view.category:
                category = self._menu_categories.get(view.category)
    
                if category is None:
                    category = MenuItem(view.category)
                    self._menu_categories[view.category] = category
                    self._menu.append(category)
    
                category.add_child(MenuItem(view.name, view))
            else:
                self._menu.append(MenuItem(view.name, view))
    
        def init_app(self, app):

            self.app = app
    
            self._init_extension()
    
            # Register views
            for view in self._views:
                app.register_blueprint(view.create_blueprint(self))
                self._add_view_to_menu(view)

从上面的代码可以看出`init_app(app)`和`Admin(app=app)`是一样的：

1. 将每个view注册为blueprint（Flask里的概念，可以简单理解为模块）
2. 记录所有view，以及所属的category和url


##AdminIndexView

    class AdminIndexView(name=None, category=None, endpoint=None, url=None, template='admin/index.html')

* name: 缺省'Home'
* endpoint: 缺省'admin' 
* url: 缺省'/admin'

如果要封装出自己的view，可以参照`AdminIndexView`的写法：

    class AdminIndexView(BaseView):
    
        def __init__(self, name=None, category=None,
                     endpoint=None, url=None,
                     template='admin/index.html'):
            super(AdminIndexView, self).__init__(name or babel.lazy_gettext('Home'),
                                                 category,
                                                 endpoint or 'admin',
                                                 url or '/admin',
                                                 'static')
            self._template = template
    
        @expose()
        def index(self):
            return self.render(self._template)


##base_template
base_template缺省是`/admin/base.html`，是页面的主要代码（基于[bootstrap](http://www.bootcss.com/)），它里面又import `admin/layout.html`；  

layout是一些宏，主要用于展开、显示menu；  

在模板中使用一些变量来取出之前注册view时保存的信息（如menu name和url等）：

    # admin/layout.html (部分) 
    {% macro menu() %}
      {% for item in admin_view.admin.menu() %}
        {% if item.is_category() %}
          {% set children = item.get_children() %}
          {% if children %}
            {% if item.is_active(admin_view) %}<li class="active dropdown">{% else %}<li class="dropdown">{% endif %}
              <a class="dropdown-toggle" data-toggle="dropdown" href="javascript:void(0)">{{ item.name }}<b class="caret"></b></a>
              <ul class="dropdown-menu">
                {% for child in children %}
                  {% if child.is_active(admin_view) %}<li class="active">{% else %}<li>{% endif %}
                    <a href="{{ child.get_url() }}">{{ child.name }}</a>
                  </li>
                {% endfor %}
              </ul>
            </li>
          {% endif %}
        {% else %}
          {% if item.is_accessible() and item.is_visible() %}
            {% if item.is_active(admin_view) %}<li class="active">{% else %}<li>{% endif %}
              <a href="{{ item.get_url() }}">{{ item.name }}</a>
            </li>
          {% endif %}
        {% endif %}
      {% endfor %}
    {% endmacro %}

#example/file
这个样例能帮助我们快速搭建起文件管理界面，但我们的重点是学习使用`ActionsMixin`模块  

`file.py`：

    import os
    import os.path as op

    from flask import Flask

    from flask.ext import admin
    from flask.ext.admin.contrib import fileadmin

    # Create flask app
    app = Flask(__name__, template_folder='templates', static_folder='files')

    # Create dummy secrey key so we can use flash
    app.config['SECRET_KEY'] = '123456790'


    # Flask views
    @app.route('/')
    def index():
        return '<a href="/admin/">Click me to get to Admin!</a>'


    if __name__ == '__main__':
        # Create directory
        path = op.join(op.dirname(__file__), 'files')
        try:
            os.mkdir(path)
        except OSError:
            pass

        # Create admin interface
        admin = admin.Admin(app)
        admin.add_view(fileadmin.FileAdmin(path, '/files/', name='Files'))

        # Start app
        app.run(debug=True)

FileAdmin是已经写好的的一个view，直接用即可：

    class FileAdmin(base_path, base_url, name=None, category=None, endpoint=None, url=None, verify_path=True)

* base_path: 文件存放的相对路径 
* base_url: 文件目录的URL

FileAdmin中和ActionsMixin相关代码如下：

    class FileAdmin(BaseView, ActionsMixin):

        def __init__(self, base_path, base_url,
                     name=None, category=None, endpoint=None, url=None,
                     verify_path=True):

            self.init_actions()

    @expose('/action/', methods=('POST',))
    def action_view(self):
        return self.handle_action()

    # Actions
    @action('delete',
            lazy_gettext('Delete'),
            lazy_gettext('Are you sure you want to delete these files?'))
    def action_delete(self, items):
        if not self.can_delete:
            flash(gettext('File deletion is disabled.'), 'error')
            return

        for path in items:
            base_path, full_path, path = self._normalize_path(path)

            if self.is_accessible_path(path):
                try:
                    os.remove(full_path)
                    flash(gettext('File "%(name)s" was successfully deleted.', name=path))
                except Exception as ex:
                    flash(gettext('Failed to delete file: %(name)s', name=ex), 'error')

    @action('edit', lazy_gettext('Edit'))
    def action_edit(self, items):
        return redirect(url_for('.edit', path=items))

`@action()`用于wrap跟在后面的函数，这里的作用就是把参数保存起来：

    def action(name, text, confirmation=None)
        def wrap(f):
            f._action = (name, text, confirmation)
            return f
    
        return wrap

* name: action name
* text: 可用于按钮名称
* confirmation: 弹框确认信息

`init_actions()`把所有action的信息保存到`ActionsMixin`里面：

    # 调试信息
    _actions = [('delete', lu'Delete'), ('edit', lu'Edit')]
    _actions_data = {'edit': (<bound method FileAdmin.action_edit of <flask_admin.contrib.fileadmin.FileAdmin object at 0x1aafc50>>, lu'Edit', None), 'delete': (<bound method FileAdmin.action_delete of <flask_admin.contrib.fileadmin.FileAdmin object at 0x1aafc50>>, lu'Delete', lu'Are you sure you want to delete these files?')}

`action_view()`用于处理POST给`/action/`的请求，然后调用`handle_action()`，它再调用不同的action处理，最后返回当前页面：

    # 省略无关代码
    def handle_action(self, return_view=None):

        action = request.form.get('action')
        ids = request.form.getlist('rowid')

        handler = self._actions_data.get(action)

        if handler and self.is_action_allowed(action):
            response = handler[0](ids)

            if response is not None:
                return response

        if not return_view:
            url = url_for('.' + self._default_view)
        else:
            url = url_for('.' + return_view)

        return redirect(url)

ids是一个文件清单，作为参数传给action处理函数（参数items）：

    # 调试信息
    ids: [u'1.png', u'2.png']

再分析页面代码，`Files`页面对应文件为`admin/file/list.html`，重点看`With selected`下拉菜单相关代码：

    {% import 'admin/actions.html' as actionslib with context %}

    {% if actions %}
        <div class="btn-group">
            {{ actionslib.dropdown(actions, 'dropdown-toggle btn btn-large') }}
        </div>
    {% endif %}

    {% block actions %}
        {{ actionslib.form(actions, url_for('.action_view')) }}
    {% endblock %}

    {% block tail %}
        {{ actionslib.script(_gettext('Please select at least one file.'),
                          actions,
                          actions_confirmation) }}
    {% endblock %}


上面用到的三个宏在`actions.html`：

    {% macro dropdown(actions, btn_class='dropdown-toggle') -%}
        <a class="{{ btn_class }}" data-toggle="dropdown" href="javascript:void(0)">{{ _gettext('With selected') }}<b class="caret"></b></a>
        <ul class="dropdown-menu">
            {% for p in actions %}
            <li>
                <a href="javascript:void(0)" onclick="return modelActions.execute('{{ p[0] }}');">{{ _gettext(p[1]) }}</a>
            </li>
            {% endfor %}
        </ul>
    {% endmacro %}

    {% macro form(actions, url) %}
        {% if actions %}
        <form id="action_form" action="{{ url }}" method="POST" style="display: none">
            {% if csrf_token %}
            <input type="hidden" name="csrf_token" value="{{ csrf_token() }}"/>
            {% endif %}
            <input type="hidden" id="action" name="action" />
        </form>
        {% endif %}
    {% endmacro %}

    {% macro script(message, actions, actions_confirmation) %}
        {% if actions %}
        <script src="{{ admin_static.url(filename='admin/js/actions.js') }}"></script>
        <script language="javascript">
            var modelActions = new AdminModelActions({{ message|tojson|safe }}, {{ actions_confirmation|tojson|safe }});
        </script>
        {% endif %}
    {% endmacro %}

最终生成的页面（部分）：

    <div class="btn-group">
        <a class="dropdown-toggle btn btn-large" href="javascript:void(0)" data-toggle="dropdown">
            With selected
            <b class="caret"></b>
        </a>

        <ul class="dropdown-menu">
            <li>
                <a onclick="return modelActions.execute('delete');" href="javascript:void(0)">Delete</a>
            </li>
            <li>
                <a onclick="return modelActions.execute('edit');" href="javascript:void(0)">Edit</a>
            </li>
        </ul>
    </div>

    <form id="action_form" action="/admin/fileadmin/action/" method="POST" style="display: none">
        <input type="hidden" id="action" name="action" />
    </form>

    <script src="/admin/static/admin/js/actions.js"></script>
    <script language="javascript">
        var modelActions = new AdminModelActions("Please select at least one file.", {"delete": "Are you sure you want to delete these files?"});
    </script>

选择菜单后的处理方法在`actions.js`:

    var AdminModelActions = function(actionErrorMessage, actionConfirmations) {
        // Actions helpers. TODO: Move to separate file
        this.execute = function(name) {
            var selected = $('input.action-checkbox:checked').size();
    
            if (selected === 0) {
                alert(actionErrorMessage);
                return false;
            }
    
            var msg = actionConfirmations[name];
    
            if (!!msg)
                if (!confirm(msg))
                    return false;
    
            // Update hidden form and submit it
            var form = $('#action_form');
            $('#action', form).val(name);
    
            $('input.action-checkbox', form).remove();
            $('input.action-checkbox:checked').each(function() {
                form.append($(this).clone());
            });
    
            form.submit();
    
            return false;
        };
    
        $(function() {
            $('.action-rowtoggle').change(function() {
                $('input.action-checkbox').attr('checked', this.checked);
            });
        });
    };

对比一下修改前后的表单：

    # 初始化
    <form id="action_form" style="display: none" method="POST" action="/admin/fileadmin/action/">
        <input id="action" type="hidden" name="action">
    </form>

    # 'Delete'选中的三个文件
    <form id="action_form" style="display: none" method="POST" action="/admin/fileadmin/action/">
        <input id="action" type="hidden" name="action" value="delete">
        <input class="action-checkbox" type="checkbox" value="1.png" name="rowid">
        <input class="action-checkbox" type="checkbox" value="2.png" name="rowid">
        <input class="action-checkbox" type="checkbox" value="3.png" name="rowid">
    </form>

    # 'Edit'选中的一个文件
    <form id="action_form" style="display: none" method="POST" action="/admin/fileadmin/action/">
        <input id="action" type="hidden" name="action" value="edit">
        <input class="action-checkbox" type="checkbox" value="1.png" name="rowid">
    </form>

总结一下，当我们点击下拉菜单中的菜单项（Delete,Edit），本地JavaScript代码会弹出确认框（假设有确认信息），然后提交一个表单给`/admin/fileadmin/action/`，请求处理函数`action_view()`根据表单类型再调用不同的action处理函数，最后返回一个页面。

{% endraw %}
