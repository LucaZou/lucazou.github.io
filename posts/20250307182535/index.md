# 【Pepper机器人管家（二)】项目组成-Web开发篇：后端

&gt;Web模块的大致结构是基于MVC架构的flask&#43;jinja2&#43;mysql的前后端不分离的结构

&lt;!--more--&gt;
# 项目概述

Web模块的大致结构是基于MVC架构的flask&#43;jinja2&#43;mysql的前后端不分离的结构，大致结构已经在上篇中介绍了，这里不做赘述了。由于项目需求比较简单而且需求特殊，后端选择自己构建导致有些用法并不完全满足falsk的开发规范。前端没有使用框架，而是使用开源项目二次开发，代码中有许多冗余。

```shell
│  README.md
│  .env //环境变量
│  app.log //运行日志
│  runserver.py //项目启动文件
│  requirements.txt //项目依赖
├─app
│  │  config.py //配置文件,数据库连接(需要在环境变量中加入此路径)
│  │  forms.py //验证表单程序
│  │  __init__.py //模块初始化文件，Flask 程序对象的创建须在 __init__.py 文件里完成
│  │  wsgi.py //wsgi服务启动
│  │
│  ├─controller //MVC中的控制器(C)
│  │     event_controller.py //事件控制器
│  │     api_controller.py //api控制器
│  │     user_controller.py //用户控制器
│  │     reminder_controller.py //提醒控制器
│  │  
│  ├─model //MVC中的模型,数据库中的表(M)
│  │     Event.py //事件表
│  │     User.py //用户表
│  │     EmailHistory //邮件历史记录
│  │
│  ├─views //MVC中的视图(V)
│  │     admin_view.py //管理员
│  │     event_view.py //事件
│  │     index_view.py //主页
│  │     user_view.py //用户
│  │     album_view.py //相册
│  │     reminder_view.py//提醒
│  │ 
│  ├─static //静态资源，css，js等
│  ├─templates //视图映射的页面
│  │      index.html //主页
│  │      create_event.html //创建事件
│  │      edit_event.html //编辑事件
│  │      event_list.html //查看事件
│  │      login.html //登录
│  │      register.html //注册
│  │      calendar.html //日历
│  │      profile.html //个人信息
│  │      album.html //相册
│  │      email_history.html //邮件页面
│  │      email_template.html //邮件模板
│  │      about.html //使用教程页面

```

## 模型-数据库的构建

项目开始项目构建数据库,也是MVC中的模型。数据库的搭建使用了flask-SQLAlchemy的orm框架，使用flask-migrate进行数据库的迁移。


**User.py:**

```python
from app import db
from werkzeug.security import generate_password_hash, check_password_hash
from flask_login import UserMixin


class User(db.Model, UserMixin):
    __tablename__ = &#39;users&#39;
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(255), nullable=False)
    email = db.Column(db.String(255), nullable=False, unique=True)
    phone = db.Column(db.String(255), nullable=False)
    password_hash = db.Column(db.String(255), nullable=False)  # 添加密码哈希字段
    is_admin = db.Column(db.Boolean, default=False, nullable=False)  # 新增 is_admin 字段

    event = db.relationship(&#39;Event&#39;, backref=&#39;user&#39;, lazy=&#39;dynamic&#39;)
    email_history = db.relationship(&#39;EmailHistory&#39;, backref=&#39;user&#39;, lazy=&#39;dynamic&#39;)

    def __init__(self, name, email, phone, password, is_admin=False):
        self.name = name
        self.email = email
        self.phone = phone
        self.set_password(password)  # 在初始化时设置密码哈希值
        self.is_admin = is_admin  # 设置用户是否是管理员

    def __repr__(self):
        return &#39;&lt;User %r&gt;&#39; % self.name

    # 添加密码哈希相关的方法
    def set_password(self, password):
        self.password_hash = generate_password_hash(password)

    def check_password(self, password):
        return check_password_hash(self.password_hash, password)

    # 实现 is_active 方法
    def is_active(self):
        return True  # 返回 True 表示用户是活动的，可以登录

```

**Event.py:**

```python
from app import db
from app.model.User import User
from sqlalchemy.ext.hybrid import hybrid_property

class Event(db.Model):
    __tablename__ = &#39;events&#39;
    id = db.Column(db.Integer, primary_key=True)
    user_id = db.Column(db.Integer, db.ForeignKey(User.id, ondelete=&#39;CASCADE&#39;), nullable=False)
    title = db.Column(db.String(255), nullable=False)
    start_date = db.Column(db.Date, nullable=False)
    event_time = db.Column(db.Time)
    end_date = db.Column(db.Date)
    location = db.Column(db.String(255))
    description = db.Column(db.String(255))
    repeat = db.Column(db.String(255), nullable=False, default=&#34;none&#34;)  # Text field for custom repeat types
    reminder_type = db.Column(db.String(255), nullable=False, default=&#34;none&#34;)
    reminder_time = db.Column(db.Float, nullable=True, default=0)

    @hybrid_property
    def duration(self):
        return self.end_date - self.start_date

    def __init__(self, user_id, title, start_date, event_time, end_date, location, description, reminder_time,
                 repeat, reminder_type):
        self.user_id = user_id
        self.title = title
        self.start_date = start_date
        self.event_time = event_time
        self.end_date = end_date
        self.location = location
        self.description = description
        self.repeat = repeat
        self.reminder_type = reminder_type
        self.reminder_time = reminder_time

    def __repr__(self):
        return &#39;&lt;Event %r&gt;&#39; % self.title

```

**EmailHistory.py:**

```python
# encoding:utf-8  
# encoding:utf-8
from datetime import datetime
from app import db
from app.model.User import User


class EmailHistory(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    user_id = db.Column(db.Integer, db.ForeignKey(User.id, ondelete=&#39;CASCADE&#39;), nullable=False)
    title = db.Column(db.String(255), nullable=False)
    content = db.Column(db.Text, nullable=False)
    recipients = db.Column(db.String(255), nullable=False)
    sent_at = db.Column(db.DateTime, default=datetime.utcnow, nullable=False)

    def __init__(self, user_id, title, content, recipients):
        self.user_id = user_id
        self.title = title
        self.content = content
        self.recipients = recipients

```

创建好SQLAlchemy后使用flask-Migrate命令初始化数据库，使用数据库迁移可以很好的管理数据库结构的改变而且不会影响数据。

### **数据库迁移**

创建迁移存储库：

```bash
flask db init
```

这会将迁移文件夹添加到应用程序中。此时，你可以发现项目目录多了一个 migrations 的文件夹，下边的 versions 目录下的文件就是生成的数据库迁移文件！

然后，运行以下命令生成迁移
```bash
flask db migrate -m &#34;initial migration&#34;
```

做完这两步就完成了第一次的初始迁移操作，我们可以看数据库已经有了我们创建的模型字段！之后，每次在新增和修改完模型数据之后，只需要执行以下两个命令即可
```bash
flask db migrate -m &#34;description of changes&#34;
flask db upgrade
```

这些步骤将允许你使用`flask_migrate`管理数据库模型的变化。请确保按照这些步骤在你的Flask应用中设置并使用`flask_migrate`，以便维护数据库模型的一致性。

使用Navicate查看数据模型：

![image.png](https://img-1313049298.cos.ap-shanghai.myqcloud.com/note-img/202503071824065.png)


## 以登录为例创建Web App



### **控制器**

用户管理使用了flask-login管理用户逻辑
我们首先编写注册和登录逻辑，创建`app/controller/user_controller.py` 。

```python
# encoding:utf-8
from flask import flash, redirect, url_for, render_template, make_response
from flask_login import login_user, login_required, logout_user, current_user
from app.forms import RegisterForm, LoginForm
from app.model.User import User
from app import db, login_manager


def register():
    &#34;&#34;&#34;
    注册新用户。

    该函数处理新用户的注册过程。它验证用户提交的表单数据，
    检查用户是否已存在，在数据库中创建新用户，并将用户重定向到登录页面
    注册成功后。

    Returns:
        如果表单数据有效并且用户注册成功，则将用户重定向到登录页面。
        否则，它会呈现带有注册表单的 register.html 模板。

    &#34;&#34;&#34;
    form = RegisterForm()
    if form.validate_on_submit():
        # 获取用户提交的数据
        name = form.name.data
        email = form.email.data
        phone = form.phone.data
        password = form.password.data

        # 检查用户是否已存在
        existing_user = User.query.filter_by(email=email).first()
        if existing_user:
            flash(&#39;该邮箱地址已被注册&#39;, &#39;error&#39;)
            return redirect(url_for(&#39;user.register&#39;))

        # 创建新用户
        new_user = User(name=name, email=email, phone=phone, password=password)
        db.session.add(new_user)
        db.session.commit()

        flash(&#39;注册成功，请登录&#39;, &#39;success&#39;)
        return redirect(url_for(&#39;user.login&#39;))
    return render_template(&#39;register.html&#39;, form=form)


def login():
    &#34;&#34;&#34;
    用户登录函数

    如果用户已登录，则重定向到受保护的页面。
    如果用户提交了有效的登录表单，则验证用户凭据并将用户标记为已登录。
    如果用户验证失败，则显示错误消息。

    return: 渲染登录页面或重定向到受保护的页面
    &#34;&#34;&#34;
    if current_user.is_authenticated:
        # 如果用户已登录，重定向到受保护的页面
        return redirect(url_for(&#39;event.show_event&#39;))
    form = LoginForm()
    if form.validate_on_submit():
        email = form.email.data  # 修改为使用邮箱作为登录凭据
        password = form.password.data
        user = User.query.filter_by(email=email).first()
        if user and user.check_password(password):
            # 用户验证成功，将用户标记为已登录
            # 可以使用 Flask-Login 或自己的会话管理逻辑来处理登录状态
            login_user(user, remember=form.remember)
            # return redirect(url_for(&#39;event.show_event&#39;))  # 跳转到用户仪表板或其他受保护的页面
            # 设置cookie
            response = make_response(redirect(url_for(&#39;event.show_event&#39;)))
            response.set_cookie(&#39;user_id&#39;, str(user.id))  # 存储用户ID在cookie中
            flash(&#39;登录成功&#39;, &#39;success&#39;)

            return response
        else:
            flash(&#39;登录失败，请检查邮箱或密码&#39;, &#39;error&#39;)

    return render_template(&#39;login.html&#39;, form=form)


@login_required
def profile():
    &#34;&#34;&#34;
    用户个人信息页面
    &#34;&#34;&#34;
    name = current_user.name
    email = current_user.email
    phone = current_user.phone

    return render_template(&#39;profile.html&#39;, name=name, email=email, phone=phone)


@login_required
def logout():
    logout_user()  # 使用 Flask-Login 注销用户
    flash(&#39;成功注销&#39;, &#39;success&#39;)
    return redirect(url_for(&#39;user.login&#39;))


@login_manager.user_loader
def load_user(user_id):
    # 使用用户 ID 查询用户对象
    return User.query.get(int(user_id))

```

**创建表单验证:**

```python
# forms.py  
from flask_wtf import FlaskForm  
from wtforms import StringField, PasswordField, validators, SubmitField, ValidationError, DateField, TimeField  
  
  
class RegisterForm(FlaskForm):  
	name = StringField(&#39;用户名&#39;, [validators.DataRequired(&#34;用户名不能为空&#34;)])  
	email = StringField(&#39;邮箱&#39;, [validators.DataRequired(&#34;邮箱不能为空&#34;), validators.Email(&#34;无效的邮箱地址&#34;)])  
	phone = StringField(&#39;电话&#39;)  
	password = PasswordField(&#39;密码&#39;, [  
	validators.DataRequired(&#34;密码不能为空&#34;),  
	validators.Length(min=8, message=&#34;密码长度至少 8 个字符&#34;)  
	])  
	confirm_password = PasswordField(&#39;确认密码&#39;, [  
	validators.DataRequired(&#34;请再次输入密码&#34;),  
	validators.EqualTo(&#39;password&#39;, message=&#39;两次密码不匹配&#39;)  
	])  
  
  
class LoginForm(FlaskForm):  
	email = StringField(&#39;邮箱&#39;, [validators.DataRequired(&#34;邮箱不能为空&#34;), validators.Email(&#34;无效的邮箱地址&#34;)])  
	password = PasswordField(&#39;密码&#39;, [validators.DataRequired(&#34;密码不能为空&#34;)])  
	remember = StringField(&#39;记住我&#39;)
```

### **视图**

项目使用flask Blueprint(蓝图)来管理路由,创建`app/views/user_view.py`

```python
# user_view.py  
from flask import Blueprint  
from app.controller import user_controller  
  
user_blueprint = Blueprint(&#39;user&#39;, __name__)  
  
user_blueprint.route(&#39;/register&#39;, methods=[&#39;GET&#39;, &#39;POST&#39;])(user_controller.register)  
user_blueprint.route(&#39;/login&#39;, methods=[&#39;GET&#39;, &#39;POST&#39;])(user_controller.login)  
user_blueprint.route(&#39;/logout&#39;, methods=[&#39;GET&#39;])(user_controller.logout)  
user_blueprint.route(&#39;/profile&#39;, methods=[&#39;GET&#39;, &#39;POST&#39;])(user_controller.profile)
```

### 将参数通过jinja2传递到前端

**注册页面:**
```html
&lt;!DOCTYPE html&gt;  
&lt;html lang=&#34;en&#34;&gt;  
&lt;head&gt;  
&lt;meta charset=&#34;utf-8&#34;&gt;  
&lt;title&gt;注册&lt;/title&gt;  
&lt;link rel=&#34;stylesheet&#34; type=&#34;text/css&#34; href=&#34;../static/css/register-login.css&#34;&gt;  
&lt;link rel=&#34;icon&#34; href=&#34;../static/images/robot.png&#34; type=&#34;image/x-icon&#34;/&gt;  
&lt;/head&gt;  
&lt;body&gt;  
&lt;div id=&#34;box&#34;&gt;&lt;/div&gt;  
&lt;div class=&#34;cent-box register-box&#34;&gt;  
&lt;div class=&#34;cent-box-header&#34;&gt;  
&lt;h1 class=&#34;main-title hide&#34;&gt;Pepper机器人&lt;/h1&gt;  
&lt;h2 class=&#34;sub-title&#34;&gt;日程管理系统&lt;/h2&gt;  
&lt;/div&gt;  
  
&lt;div class=&#34;cont-main clearfix&#34;&gt;  
&lt;div class=&#34;index-tab&#34;&gt;  
&lt;div class=&#34;index-slide-nav&#34;&gt;  
&lt;a href=&#34;{{ url_for(&#39;user.login&#39;) }}&#34;&gt;登录&lt;/a&gt;  
&lt;a href=&#34;{{ url_for(&#39;user.register&#39;) }}&#34; class=&#34;active&#34;&gt;注册&lt;/a&gt;  
&lt;div class=&#34;slide-bar slide-bar1&#34;&gt;&lt;/div&gt;  
&lt;/div&gt;  
&lt;/div&gt;  
&lt;form method=&#34;POST&#34;&gt;  
{{ form.hidden_tag() }}  
&lt;div class=&#34;login form&#34;&gt;  
&lt;div class=&#34;group&#34;&gt;  
&lt;div class=&#34;group-ipt name&#34;&gt;  
&lt;label for=&#34;name&#34;&gt;&lt;/label&gt;  
{% if form.name.data is not none %}  
&lt;input type=&#34;text&#34; name=&#34;name&#34; id=&#34;name&#34; class=&#34;ipt&#34; value=&#34;{{ form.name.data }}&#34; placeholder=&#34;输入您的用户名&#34; required&gt;  
{% else %}  
&lt;input type=&#34;text&#34; name=&#34;name&#34; id=&#34;name&#34; class=&#34;ipt&#34; placeholder=&#34;输入您的用户名&#34; required&gt;  
{% endif %}  
&lt;/div&gt;  
&lt;div class=&#34;group-ipt email&#34;&gt;  
&lt;label for=&#34;email&#34;&gt;&lt;/label&gt;  
{% if form.email.data is not none %}  
&lt;input type=&#34;email&#34; name=&#34;email&#34; id=&#34;email&#34; class=&#34;ipt&#34; value=&#34;{{ form.email.data }}&#34; placeholder=&#34;请输入您的邮箱&#34; required&gt;  
{% else %}  
&lt;input type=&#34;email&#34; name=&#34;email&#34; id=&#34;email&#34; class=&#34;ipt&#34; placeholder=&#34;请输入您的邮箱&#34; required&gt;  
{% endif %}  
&lt;/div&gt;  
&lt;div class=&#34;group-ipt phone&#34;&gt;  
&lt;label for=&#34;phone&#34;&gt;&lt;/label&gt;  
{% if form.phone.data is not none %}  
&lt;input type=&#34;text&#34; name=&#34;phone&#34; id=&#34;phone&#34; class=&#34;ipt&#34; value=&#34;{{ form.phone.data }}&#34; placeholder=&#34;输入您的电话&#34; required&gt;  
{% else %}  
&lt;input type=&#34;text&#34; name=&#34;phone&#34; id=&#34;phone&#34; class=&#34;ipt&#34; placeholder=&#34;输入您的电话&#34; required&gt;  
{% endif %}  
&lt;/div&gt;  
&lt;div class=&#34;group-ipt password&#34;&gt;  
&lt;label for=&#34;password&#34;&gt;&lt;/label&gt;&lt;input type=&#34;password&#34; name=&#34;password&#34; id=&#34;password&#34; class=&#34;ipt&#34; placeholder=&#34;输入密码&#34; required&gt;  
&lt;/div&gt;  
&lt;div class=&#34;group-ipt confirm_password&#34;&gt;  
&lt;label for=&#34;confirm_password&#34;&gt;&lt;/label&gt;&lt;input type=&#34;password&#34; name=&#34;confirm_password&#34; id=&#34;confirm_password&#34; class=&#34;ipt&#34; placeholder=&#34;确认密码&#34; required&gt;  
&lt;/div&gt;  
  
&lt;/div&gt;  
&lt;div class=&#34;button&#34;&gt;  
&lt;button type=&#34;submit&#34; class=&#34;register-btn&#34; id=&#34;button&#34; value=&#34;注册&#34;&gt;注册&lt;/button&gt;  
&lt;/div&gt;  
&lt;/div&gt;  
&lt;/form&gt;  
  
&lt;/div&gt;  
&lt;/div&gt;  
  
&lt;script src=&#39;../static/js/particles.js&#39; type=&#34;text/javascript&#34;&gt;&lt;/script&gt;  
&lt;script src=&#39;../static/js/background.js&#39; type=&#34;text/javascript&#34;&gt;&lt;/script&gt;  
&lt;script src=&#39;../static/js/jquery.min.js&#39; type=&#34;text/javascript&#34;&gt;&lt;/script&gt;  
&lt;script src=&#39;../static/js/layer/layer.js&#39; type=&#34;text/javascript&#34;&gt;&lt;/script&gt;  
&lt;script src=&#39;../static/js/index.js&#39; type=&#34;text/javascript&#34;&gt;&lt;/script&gt;  
&lt;/body&gt;  
&lt;/html&gt;
```

效果图:
![image.png](https://img-1313049298.cos.ap-shanghai.myqcloud.com/note-img/202503071824066.png)


**登录页面:**

```html
&lt;!DOCTYPE html&gt;  
&lt;html lang=&#34;en&#34;&gt;  
&lt;head&gt;  
&lt;meta charset=&#34;utf-8&#34;&gt;  
&lt;title&gt;登录&lt;/title&gt;  
&lt;link rel=&#34;stylesheet&#34; type=&#34;text/css&#34; href=&#34;{{ url_for(&#39;static&#39;, filename=&#39;css/register-login.css&#39;) }}&#34;&gt;  
&lt;link rel=&#34;icon&#34; href=&#34;{{ url_for(&#39;static&#39;, filename=&#39;images/robot.png&#39;) }}&#34; type=&#34;image/x-icon&#34;/&gt;  
&lt;/head&gt;  
&lt;body&gt;  
&lt;div id=&#34;box&#34;&gt;&lt;/div&gt;  
&lt;div class=&#34;cent-box&#34;&gt;  
&lt;div class=&#34;cent-box-header&#34;&gt;  
&lt;h1 class=&#34;main-title hide&#34;&gt;Pepper机器人&lt;/h1&gt;  
&lt;h2 class=&#34;sub-title&#34;&gt;日程管理系统&lt;/h2&gt;  
&lt;/div&gt;  
  
&lt;div class=&#34;cont-main clearfix&#34;&gt;  
&lt;div class=&#34;index-tab&#34;&gt;  
&lt;div class=&#34;index-slide-nav&#34;&gt;  
&lt;a href=&#34;{{ url_for(&#39;user.login&#39;) }}&#34; class=&#34;active&#34;&gt;登录&lt;/a&gt;  
&lt;a href=&#34;{{ url_for(&#39;user.register&#39;) }}&#34;&gt;注册&lt;/a&gt;  
&lt;div class=&#34;slide-bar&#34;&gt;&lt;/div&gt;  
&lt;/div&gt;  
&lt;/div&gt;  
&lt;form method=&#34;POST&#34;&gt;  
{{ form.hidden_tag() }}  
&lt;div class=&#34;login form&#34;&gt;  
&lt;div class=&#34;group&#34;&gt;  
&lt;div class=&#34;group-ipt email&#34;&gt;  
&lt;label for=&#34;email&#34;&gt;&lt;/label&gt;  
{% if form.email.data is not none %}  
&lt;input type=&#34;email&#34; name=&#34;email&#34; id=&#34;email&#34; class=&#34;ipt&#34; value=&#34;{{ form.email.data }}&#34; placeholder=&#34;输入您的邮箱&#34; required&gt;  
{% else %}  
&lt;input type=&#34;email&#34; name=&#34;email&#34; id=&#34;email&#34; class=&#34;ipt&#34; placeholder=&#34;输入您的邮箱&#34; required&gt;  
{% endif %}  
&lt;/div&gt;  
&lt;div class=&#34;group-ipt password&#34;&gt;  
&lt;label for=&#34;password&#34;&gt;&lt;/label&gt;&lt;input type=&#34;password&#34; name=&#34;password&#34; id=&#34;password&#34; class=&#34;ipt&#34; placeholder=&#34;输入您的登录密码&#34; required&gt;  
&lt;/div&gt;  
&lt;/div&gt;  
&lt;div class=&#34;button&#34;&gt;  
&lt;button type=&#34;submit&#34; class=&#34;login-btn register-btn&#34; id=&#34;button&#34; value=&#34;登录&#34;&gt;登录&lt;/button&gt;  
&lt;/div&gt;  
&lt;div class=&#34;remember clearfix&#34;&gt;  
&lt;label class=&#34;remember-me&#34;&gt;&lt;input type=&#34;checkbox&#34; name=&#34;remember&#34;&gt; 记住我&lt;/label&gt;  
&lt;/div&gt;  
&lt;/div&gt;  
&lt;/form&gt;  
&lt;/div&gt;  
&lt;/div&gt;  
  
&lt;script src=&#39;../static/js/particles.js&#39; type=&#34;text/javascript&#34;&gt;&lt;/script&gt;  
&lt;script src=&#39;../static/js/background.js&#39; type=&#34;text/javascript&#34;&gt;&lt;/script&gt;  
&lt;script src=&#39;../static/js/jquery.min.js&#39; type=&#34;text/javascript&#34;&gt;&lt;/script&gt;  
&lt;script src=&#39;../static/js/layer/layer.js&#39; type=&#34;text/javascript&#34;&gt;&lt;/script&gt;  
&lt;script src=&#39;../static/js/index.js&#39; type=&#34;text/javascript&#34;&gt;&lt;/script&gt;  
&lt;/body&gt;  
&lt;/html&gt;
```

效果图:

![image.png](https://img-1313049298.cos.ap-shanghai.myqcloud.com/note-img/202503071824067.png)

### 配置config.py

```python
# 调试模式是否开启  
DEBUG = True  
# 是否追踪对象的修改。  
SQLALCHEMY_TRACK_MODIFICATIONS = True  
# 查询时显示原始SQL语句  
SQLALCHEMY_ECHO = False  
# session必须要设置key  
SECRET_KEY = &#39;c798ee1f5fd894f6e0ba9fc0d16b8b22&#39;  
# mysql数据库连接信息  
DATABASE = &#39;schedule&#39;  
SQLALCHEMY_DATABASE_URI = &#34;mysql&#43;pymysql://[user]:[password]@localhost/&#34; &#43; DATABASE
```

### 使用工厂函数初始化

`app/_init_.py`

```python
from flask import Flask  
from flask_sqlalchemy import SQLAlchemy  
from flask_login import LoginManager  
from flask_migrate import Migrate  
from flask_restful import Api  
from flask_admin import Admin  
from flask_mail import Mail  
from flask_uploads import configure_uploads  
import logging  
from logging.handlers import RotatingFileHandler  
from flask_apscheduler import APScheduler  
  
  
# 创建Flask应用实例  
db = SQLAlchemy() # 数据库实例  
login_manager = LoginManager() # 登录管理实例  
migrate = Migrate() # 数据库迁移实例   
  
  
def create_app() -&gt; Flask:  
&#34;&#34;&#34;创建Flask应用实例并进行初始化设置。  
  
Returns:  
Flask: 初始化设置后的Flask应用实例。  
&#34;&#34;&#34;  
	app = Flask(__name__)  
	app.config.from_object(&#39;app.config&#39;)  
	app.config.from_envvar(&#39;FLASKR_CONFIGS&#39;)  
	  
	configure_logging(app) # 添加日志配置  
	initialize_extensions(app) # 初始化  
	register_blueprints(app) # 注册蓝图  
	
	return app  
  
def initialize_extensions(app: Flask) -&gt; None:  
&#34;&#34;&#34;初始化扩展，包括数据库、登录管理、数据库迁移、邮件和文件上传。  
  
Args:  
app (Flask): Flask应用实例。  
&#34;&#34;&#34;  
	db.init_app(app)  
	migrate.init_app(app, db)  
	login_manager.init_app(app)  
	login_manager.login_view = &#39;user.login&#39;  
  
  
def register_blueprints(app: Flask) -&gt; None:  
&#34;&#34;&#34;注册蓝图，包括主页、用户、事件和提醒蓝图。  
  
Args:  
app (Flask): Flask应用实例。  
&#34;&#34;&#34;   
	from app.views.user_view import user_blueprint  
	
	app.register_blueprint(user_blueprint, url_prefix=&#39;/user&#39;)  


def configure_logging(app: Flask) -&gt; None:  
&#34;&#34;&#34;配置应用程序日志。  
  
Args:  
app (Flask): Flask应用实例。  
&#34;&#34;&#34;  
	log_file_path = &#39;app.log&#39;  
	log_handler = RotatingFileHandler(log_file_path, maxBytes=10240, backupCount=10)  
	log_handler.setFormatter(logging.Formatter(  
	&#39;%(asctime)s %(levelname)s: %(message)s &#39;  
	&#39;[in %(pathname)s:%(lineno)d]&#39;  
	))  
	log_handler.setLevel(logging.INFO)  
	app.logger.addHandler(log_handler)  
  
```

### 运行项目

`app/runserver.py`

```python
from app import create_app  
  
if __name__ == &#39;__main__&#39;:  
	app = create_app()  
	app.run(host=&#39;0.0.0.0&#39;, port=5000, debug=True)
```


---

> 作者:   
> URL: https://lucazou.github.io/posts/20250307182535/  

