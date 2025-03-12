# 【Pepper机器人管家（三）】项目组成-Web开发篇-日程逻辑的创建

&gt;上篇已经以用户注册登录为例创建了一个完整的demo，这篇继续在原有的基础上添加日程管理模块的代码，这也是整个项目中最核心的代码。

&lt;!--more--&gt;
# 前言

上篇已经以用户注册登录为例创建了一个完整的demo，这篇继续在原有的基础上添加日程管理模块的代码，这也是整个项目中最核心的代码。

# 日程管理的实现

按照MVC的模式一般是:模型--&gt;视图--&gt;控制器的顺序
![image.png](https://img-1313049298.cos.ap-shanghai.myqcloud.com/note-img/202503071826518.png)


## 创建模型

模型的创建其实在上篇中已经完成:

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


## 创建视图

考虑到业务需求,主要是对Event表进行CRUD操作,所以在`event`蓝图下创建了四个路由:
- /event_list
- /creat_event
- /delete_event/&lt;int:event_id&gt;
- /edit_event/&lt;int:event_id&gt;

创建`app/views/event_view.py`

```python
# event_view.py  
from flask import Blueprint  
from app.controller import event_controller # 导入 event_controller 模块  
  
event_blueprint = Blueprint(&#39;event&#39;, __name__)  
  
# 将视图函数与蓝图关联  
event_blueprint.route(&#39;/event_list&#39;)(event_controller.show_event)  
event_blueprint.route(&#39;/create_event&#39;, methods=[&#39;GET&#39;, &#39;POST&#39;])(event_controller.create_event)  
event_blueprint.route(&#39;/delete_event/&lt;int:event_id&gt;&#39;, methods=[&#39;POST&#39;])(event_controller.delete_event)  
event_blueprint.route(&#39;/edit_event/&lt;int:event_id&gt;&#39;, methods=[&#39;GET&#39;, &#39;POST&#39;])(event_controller.edit_event)  
```


## 创建控制器

明确了视图,现在只需要完成对Event表的CRUD即可，创建`app/controller/event_controller.py`

### 查询事件

使用flask-login插件管理用户,利用@login_required装饰器获取当前登录用户的id。创建一个动态生成事件的函数`get_user_events`按照用户设置的规则动态生成将要在前端显示的事件,这样可以够支持用户设置特点频率的事件而且能避免数据库的负担过大。编写一个逻辑检查过期的事件并同未过期的事件一并返回。

```python
@login_required
def show_event():
    user_id = current_user.id  # 获取当前登录用户的ID
    today_events, expired_events = get_user_events(user_id)
    return render_template(&#39;event_list.html&#39;, events=today_events, expired_events=expired_events)
def get_user_events(user_id):
    &#34;&#34;&#34;
    查询当前用户的所有事件，并根据事件的重复规则生成今天的事件列表和过期事件列表。

    Args:
        user_id (int): 用户ID

    Returns:
        tuple: 包含今天的事件列表和过期事件列表的元组
    &#34;&#34;&#34;
    # 查询当前用户的所有事件
    events = Event.query.filter_by(user_id=user_id).all()
    today = datetime.now().date()  # 获取当前日期
    today_events = []
    expired_events = []
    for event in events:
        # 如果事件过期，将其添加到过期事件列表中
        if event.end_date &lt; today:
            expired_events.append(event)
            continue
        # 如果事件不是重复事件，将其添加到今天事件列表中
        if event.repeat == &#34;none&#34; and today &lt;= event.end_date:
            today_events.append(event)
        # 如果事件是重复事件，动态生成后将其添加到今天事件列表中
        else:
            start_date = event.start_date
            end_date = event.end_date

            if event.repeat == &#34;daily&#34;:
                current_date = start_date
                while current_date &lt;= end_date:
                    if current_date &gt;= today:
                        new_event = generate_repeat_event(event, current_date)
                        today_events.append(new_event)
                        break
                    current_date &#43;= timedelta(days=1)
            elif event.repeat == &#34;weekly&#34;:
                if end_date-today &lt; timedelta(days=7):
                    expired_events.append(event)
                    continue
                current_date = start_date
                while current_date &lt;= end_date:
                    if current_date &gt;= today:
                        last_week_event = generate_repeat_event(event, current_date)
                        today_events.append(last_week_event)
                        break
                    current_date &#43;= timedelta(weeks=1)
            elif event.repeat == &#34;monthly&#34;:
                if end_date-today &lt; timedelta(days=30):
                    expired_events.append(event)
                    continue
                current_date = start_date
                while current_date &lt;= end_date:
                    if current_date &gt;= today:
                        last_month_event = generate_repeat_event(event, current_date)
                        today_events.append(last_month_event)
                        break
                    current_date &#43;= timedelta(days=30)
    return today_events, expired_events

# 创建一个函数，用于生成重复事件
def generate_repeat_event(event, current_date):
    new_event = Event(
        user_id=event.user_id,
        title=event.title,
        start_date=current_date,
        end_date=current_date,
        event_time=event.event_time,
        location=event.location,
        description=event.description,
        repeat=event.repeat,
        reminder_type=event.reminder_type,
        reminder_time=event.reminder_time
    )
    new_event.id = event.id  # 设置新事件的ID为原事件的ID
    return new_event```

### 创建事件

在用户提交表单之前,使用表单验证能够有效规范数据库在forms.py上加入对Event的表单验证:

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


class EventForm(FlaskForm):
    title = StringField(&#39;标题&#39;, validators=[validators.DataRequired(), validators.Length(max=255)])
    start_date = DateField(&#39;开始日期&#39;, format=&#39;%Y-%m-%d&#39;, validators=[validators.DataRequired()])
    event_time = TimeField(&#39;事件时间&#39;, format=&#39;%H:%M&#39;)
    end_date = DateField(&#39;结束时间&#39;, format=&#39;%Y-%m-%d&#39;)
    location = StringField(&#39;地点&#39;, validators=[validators.Length(max=255)])
    description = StringField(&#39;描述&#39;, validators=[validators.Length(max=255)])
    repeat = StringField(&#39;重复&#39;)
    reminder_type = StringField(&#39;提醒类型&#39;, validators=[validators.DataRequired()])
    reminder_time = StringField(&#39;提醒时间&#39;)
    submit = SubmitField(&#39;Submit&#39;)  # 添加这一行来定义 submit 字段

    # 如果结束时间小于或等于开始时间，则引发验证错误。
    def validate_end_time(self, field):
        if field.data &lt;= self.start_date.data:
            raise ValidationError(&#39;End time must be greater than start time&#39;)

```

编写创建事件逻辑:

```python
@login_required
def create_event():
    &#34;&#34;&#34;
    创建事件并保存到数据库。

    return: 重定向到事件列表页面或其他适当的页面
    &#34;&#34;&#34;
    form = EventForm()  # 使用您的事件表单来接收用户输入
    if not form.validate_on_submit():
        for field, errors in form.errors.items():
            for error in errors:
                print(f&#34;字段 {field} 中的错误: {error}&#34;)
    if form.validate_on_submit():
        # 获取用户输入的事件信息
        title = form.title.data
        event_time = form.event_time.data
        start_date = form.start_date.data
        end_date = form.end_date.data
        location = form.location.data
        description = form.description.data
        repeat = form.repeat.data  # 从表单获取重复类型
        reminder_type = form.reminder_type.data
        reminder_time = form.reminder_time.data

        # 创建事件对象并保存到数据库
        event = Event(
            user_id=current_user.id,  # 使用当前登录用户的 ID
            title=title,
            event_time=event_time,
            start_date=start_date,
            end_date=end_date,
            location=location,
            description=description,
            repeat=repeat,  # 设置事件的重复类型
            reminder_type=reminder_type,
            reminder_time=reminder_time
        )
        db.session.add(event)
        db.session.commit()

        flash(&#39;事件创建成功&#39;, &#39;success&#39;)

        return redirect(url_for(&#39;event.show_event&#39;))  # 重定向到事件列表页面或其他适当的页面

    return render_template(&#39;create_event.html&#39;, form=form)
```

### 删除事件

```python
@login_required
def delete_event(event_id):
    # 查询要删除的事件
    event = Event.query.get(event_id)

    # 检查事件是否存在并属于当前用户
    if event and event.user_id == current_user.id:
        db.session.delete(event)
        db.session.commit()
        flash(&#39;事件删除成功&#39;, &#39;success&#39;)
    else:
        flash(&#39;无法删除事件&#39;, &#39;error&#39;)

    return redirect(url_for(&#39;event.show_event&#39;))

```

### 编辑事件

```python
@login_required
def edit_event(event_id):
    # 查询要编辑的事件
    event = Event.query.get(event_id)

    # 检查事件是否存在并属于当前用户
    if not event or event.user_id != current_user.id:
        flash(&#39;无法编辑事件&#39;, &#39;error&#39;)
        return redirect(url_for(&#39;event.show_event&#39;))

    form = EventForm(obj=event)  # 使用事件对象填充表单

    if form.validate_on_submit():
        # 更新事件信息
        event.title = form.title.data
        event.event_time = form.event_time.data
        event.start_date = form.start_date.data
        event.end_date = form.end_date.data
        event.location = form.location.data
        event.description = form.description.data
        event.repeat = form.repeat.data  # 从表单获取重复类型
        event.reminder_type = form.reminder_type.data
        event.reminder_time = form.reminder_time.data

        db.session.commit()
        flash(&#39;事件更新成功&#39;, &#39;success&#39;)
        return redirect(url_for(&#39;event.show_event&#39;))

    return render_template(&#39;edit_event.html&#39;, form=form)

```

# 前端页面

完成了后端的逻辑,接下来只需要通过jinja2语法创建模板即可。由于html代码可读性差，以下使用项目的初始demo来说明。demo中只有jinja2接口，基本上没有任何样式，但是代码可读性强。

## 构建 `/event/event_list`对应的页面

创建模板`app/templates/event_list.html`

```html
&lt;!DOCTYPE html&gt;
&lt;html lang=&#34;en&#34;&gt;
&lt;head&gt;
    &lt;title&gt;事件列表&lt;/title&gt;
&lt;/head&gt;
&lt;body&gt;
    &lt;h1&gt;事件列表&lt;/h1&gt;
    &lt;ul&gt;
    &lt;!--显示待执行的事件--&gt;
        {% for event in events %}
            &lt;li&gt;
                &lt;h2&gt;{{ event.title }}&lt;/h2&gt;
                &lt;p&gt;&lt;strong&gt;日期：&lt;/strong&gt;{{ event.start_date }} - {{ event.end_date }}&lt;/p&gt;
                &lt;p&gt;&lt;strong&gt;时间：&lt;/strong&gt;{{ event.event_time }}&lt;/p&gt;
                &lt;p&gt;&lt;strong&gt;地点：&lt;/strong&gt;{{ event.location }}&lt;/p&gt;
                &lt;p&gt;&lt;strong&gt;描述：&lt;/strong&gt;{{ event.description }}&lt;/p&gt;
                &lt;p&gt;&lt;strong&gt;重复：&lt;/strong&gt;{{ event.repeat }}&lt;/p&gt;
                &lt;p&gt;&lt;strong&gt;提醒类型:&lt;/strong&gt;{{ event.reminder_type }}&lt;/p&gt;
                {% with days=event.reminder_time // (24 * 60), hours=(event.reminder_time % (24 * 60)) // 60, minutes=(event.reminder_time % 60) %}
                    &lt;p&gt;&lt;strong&gt;提醒时间:&lt;/strong&gt;{{ days }} 天 {{ hours }} 小时 {{ minutes }} 分钟&lt;/p&gt;
                {% endwith %}
                &lt;a href=&#34;{{ url_for(&#39;event.edit_event&#39;, event_id=event.id) }}&#34;&gt;编辑&lt;/a&gt;
                &lt;form method=&#34;post&#34; action=&#34;{{ url_for(&#39;event.delete_event&#39;, event_id=event.id) }}&#34;&gt;
                    &lt;input type=&#34;submit&#34; value=&#34;删除&#34;&gt;
                &lt;/form&gt;
            &lt;/li&gt;
        {% endfor %}
    &lt;!--显示过期的事件--&gt;
        {% for event in expired_events %}
            &lt;li style=&#34;color: gray&#34;&gt;
                &lt;h2&gt;{{ event.title }}&lt;/h2&gt;
                &lt;p style=&#34;color: red&#34;&gt;&lt;strong&gt;日期：&lt;/strong&gt;{{ event.start_date }} - {{ event.end_date }}&lt;/p&gt;
                &lt;p&gt;&lt;strong&gt;时间：&lt;/strong&gt;{{ event.event_time }}&lt;/p&gt;
                &lt;p&gt;&lt;strong&gt;地点：&lt;/strong&gt;{{ event.location }}&lt;/p&gt;
                &lt;p&gt;&lt;strong&gt;描述：&lt;/strong&gt;{{ event.description }}&lt;/p&gt;
                &lt;p&gt;&lt;strong&gt;重复：&lt;/strong&gt;{{ event.repeat }}&lt;/p&gt;
                &lt;p&gt;&lt;strong&gt;提醒类型:&lt;/strong&gt;{{ event.reminder_type }}&lt;/p&gt;
                {% with days=event.reminder_time // (24 * 60), hours=(event.reminder_time % (24 * 60)) // 60, minutes=(event.reminder_time % 60) %}
                    &lt;p&gt;&lt;strong&gt;提醒时间:&lt;/strong&gt;{{ days }} 天 {{ hours }} 小时 {{ minutes }} 分钟&lt;/p&gt;
                {% endwith %}
                &lt;a href=&#34;{{ url_for(&#39;event.edit_event&#39;, event_id=event.id) }}&#34;&gt;编辑&lt;/a&gt;
                &lt;form method=&#34;post&#34; action=&#34;{{ url_for(&#39;event.delete_event&#39;, event_id=event.id) }}&#34;&gt;
                    &lt;input type=&#34;submit&#34; value=&#34;删除&#34;&gt;
                &lt;/form&gt;
            &lt;/li&gt;
    {% endfor %}
    &lt;/ul&gt;
    &lt;a href=&#34;{{ url_for(&#39;event.create_event&#39;) }}&#34;&gt;创建事件&lt;/a&gt;
&lt;/body&gt;
&lt;/html&gt;

```

## 构建 `/event/creat_event` 对应的页面

创建模板 `app/templates/create_event.html`，注意需要在提交表单前加上`{{ form.hidden_tag() }}`保证通过表单验证

```html
&lt;!DOCTYPE html&gt;
&lt;html lang=&#34;en&#34;&gt;
&lt;head&gt;
    &lt;title&gt;Create Event&lt;/title&gt;
    &lt;link rel=&#34;stylesheet&#34; href=&#34;{{ url_for(&#39;static&#39;, filename=&#39;css/http_cdn.jsdelivr.net_npm_flatpickr_dist_flatpickr.css&#39;) }}&#34;&gt;
    &lt;script src=&#34;{{ url_for(&#39;static&#39;, filename=&#39;js/http_cdn.jsdelivr.net_npm_flatpickr.js&#39;) }}&#34;&gt;&lt;/script&gt;
&lt;/head&gt;
&lt;body&gt;
    &lt;h1&gt;Create Event&lt;/h1&gt;
    &lt;form onsubmit=&#34;combineAndSubmit()&#34; method=&#34;POST&#34;&gt;
        {{ form.hidden_tag() }}
        &lt;div&gt;
            {{ form.title.label(class=&#34;form-label&#34;) }}
            {{ form.title(class=&#34;form-control&#34;) }}
            {% for error in form.title.errors %}
                &lt;span style=&#34;color: red;&#34;&gt;{{ error }}&lt;/span&gt;
            {% endfor %}
        &lt;/div&gt;
        &lt;div&gt;
            {{ form.start_date.label(class=&#34;form-label&#34;) }}
            {{ form.start_date(class=&#34;form-control flatpickr&#34;,id=&#34;start_date&#34;) }}
            {% for error in form.start_date.errors %}
                &lt;span style=&#34;color: red;&#34;&gt;{{ error }}&lt;/span&gt;
            {% endfor %}
        &lt;/div&gt;
            &lt;div&gt;
            {{ form.event_time.label(class=&#34;form-label&#34;) }}
            {{ form.event_time(class=&#34;form-control flatpickr&#34;,id=&#34;event_time&#34;) }}
            {% for error in form.event_time.errors %}
                &lt;span style=&#34;color: red;&#34;&gt;{{ error }}&lt;/span&gt;
            {% endfor %}
        &lt;/div&gt;
        &lt;div&gt;
            {{ form.end_date.label(class=&#34;form-label&#34;) }}
            {{ form.end_date(class=&#34;form-control, flatpickr&#34;,id=&#34;end_date&#34;) }}
            {% for error in form.end_date.errors %}
                &lt;span style=&#34;color: red;&#34;&gt;{{ error }}&lt;/span&gt;
            {% endfor %}
        &lt;/div&gt;
        &lt;div&gt;
            {{ form.location.label(class=&#34;form-label&#34;) }}
            {{ form.location(class=&#34;form-control&#34;) }}
            {% for error in form.location.errors %}
                &lt;span style=&#34;color: red;&#34;&gt;{{ error }}&lt;/span&gt;
            {% endfor %}
        &lt;/div&gt;
        &lt;div&gt;
            {{ form.description.label(class=&#34;form-label&#34;) }}
            {{ form.description(class=&#34;form-control&#34;) }}
            {% for error in form.description.errors %}
                &lt;span style=&#34;color: red;&#34;&gt;{{ error }}&lt;/span&gt;
            {% endfor %}
        &lt;/div&gt;
        &lt;div&gt;
            &lt;label for=&#34;repeatSelect&#34;&gt;{{ form.repeat.label(class=&#34;form-label&#34;) }}&lt;/label&gt;
            &lt;select name=&#34;repeat&#34; id=&#34;repeatSelect&#34; class=&#34;form-control&#34;&gt;
                &lt;option value=&#34;none&#34;&gt;无&lt;/option&gt;
                &lt;option value=&#34;daily&#34;&gt;每天&lt;/option&gt;
                &lt;option value=&#34;weekly&#34;&gt;每周&lt;/option&gt;
                &lt;option value=&#34;month&#34;&gt;每月&lt;/option&gt;
            &lt;/select&gt;
            {% for error in form.repeat.errors %}
                &lt;span style=&#34;color: red;&#34;&gt;{{ error }}&lt;/span&gt;
            {% endfor %}
        &lt;/div&gt;
        &lt;div&gt;
            &lt;label for=&#34;reminder_type&#34;&gt;{{ form.reminder_type.label(class=&#34;form-label&#34;) }}&lt;/label&gt;
            &lt;select name=&#34;reminder_type&#34; id=&#34;reminder_type&#34; class=&#34;form-control&#34;&gt;
                &lt;option value=&#34;none&#34;&gt;无&lt;/option&gt;
                &lt;option value=&#34;email&#34;&gt;邮箱&lt;/option&gt;
                &lt;option value=&#34;phone&#34;&gt;手机&lt;/option&gt;
                &lt;option value=&#34;pepper&#34;&gt;机器人&lt;/option&gt;
            &lt;/select&gt;
            {% for error in form.reminder_type.errors %}
                &lt;span style=&#34;color: red;&#34;&gt;{{ error }}&lt;/span&gt;
            {% endfor %}
        &lt;/div&gt;
        &lt;div&gt;
            {{ form.reminder_time.label(class=&#34;form-label&#34;) }}
             &lt;input type=&#34;button&#34; value=&#34;设置提醒&#34; onclick=&#34;showReminderOptions()&#34;&gt;
            {% for error in form.reminder_time.errors %}
                &lt;span style=&#34;color: red;&#34;&gt;{{ error }}&lt;/span&gt;
            {% endfor %}
        &lt;/div&gt;
        &lt;div id=&#34;reminderOptions&#34; style=&#34;display: none;&#34;&gt;
        &lt;label for=&#34;daysInput&#34;&gt;天数：&lt;/label&gt;
        &lt;input type=&#34;number&#34; id=&#34;daysInput&#34; name=&#34;daysInput&#34; min=&#34;0&#34;&gt;
        &lt;br&gt;
        &lt;label for=&#34;hoursInput&#34;&gt;小时：&lt;/label&gt;
        &lt;select id=&#34;hoursInput&#34; name=&#34;hoursInput&#34;&gt;
            &lt;!-- 使用 JavaScript 动态生成选项 --&gt;
        &lt;/select&gt;
        &lt;br&gt;
        &lt;label for=&#34;minutesInput&#34;&gt;分钟：&lt;/label&gt;
        &lt;select id=&#34;minutesInput&#34; name=&#34;minutesInput&#34;&gt;
            &lt;!-- 使用 JavaScript 动态生成选项 --&gt;
        &lt;/select&gt;
        &lt;/div&gt;
        &lt;div&gt;
        &lt;!-- 隐藏的input元素用于存储提醒时间 --&gt;
            &lt;input type=&#34;hidden&#34; name=&#34;reminder_time&#34; id=&#34;reminder_time&#34; value=&#34;&#34;&gt;
        &lt;/div&gt;
        &lt;div&gt;
            {{ form.submit()}}
        &lt;/div&gt;
    &lt;/form&gt;
&lt;script&gt;
    flatpickr(&#39;#start_date&#39;, {
        enableTime: false,
        dateFormat: &#34;Y-m-d&#34;,
    });
    flatpickr(&#39;#event_time&#39;, {
        enableTime: true,
        noCalendar: true, // 禁用日期选择
        time_24hr: true, // 使用24小时制
        dateFormat: &#34;H:i&#34;,
    });
    flatpickr(&#39;#end_date&#39;, {
        enableTime: false,
        dateFormat: &#34;Y-m-d&#34;,
    });
&lt;/script&gt;
&lt;script&gt;
    function populateOptions(element, max) {
        for (let i = 0; i &lt;= max; i&#43;&#43;) {
            const option = document.createElement(&#39;option&#39;);
            option.value = i;
            option.text = i;
            element.appendChild(option);
        }
    }

    // 动态生成小时和分钟的选项
    populateOptions(document.getElementById(&#39;hoursInput&#39;), 24);
    populateOptions(document.getElementById(&#39;minutesInput&#39;), 60);

    function showReminderOptions() {
        // 显示设置提醒时间的选项
        document.getElementById(&#39;reminderOptions&#39;).style.display = &#39;block&#39;;
    }

    function combineAndSubmit() {
        var daysInput = parseFloat(document.getElementById(&#39;daysInput&#39;).value) || 0;
        var hoursInput = parseFloat(document.getElementById(&#39;hoursInput&#39;).value) || 0;
        var minutesInput = parseFloat(document.getElementById(&#39;minutesInput&#39;).value) || 0;

        // 转换为分钟
        var totalMinutes = daysInput * 24 * 60 &#43; hoursInput * 60 &#43; minutesInput;

        // 将totalMinutes设置到表单的form.reminder_time字段
        document.getElementById(&#39;reminder_time&#39;).value = totalMinutes;

        // 可以将表单提交到服务器或进行其他操作
        alert(&#34;总时间（分钟）：&#34; &#43; totalMinutes);
    }
&lt;/script&gt;
&lt;/body&gt;
&lt;/html&gt;
```

## 构建 `/event/edit_event/` 对应的页面

创建模板 `app/templates/edit_event.html`,`/event/delete_event/` 路由没有页面,而是通过按钮来触发并删除事件

```html
&lt;!DOCTYPE html&gt;
&lt;html lang=&#34;en&#34;&gt;
&lt;head&gt;
    &lt;title&gt;Create Event&lt;/title&gt;
    &lt;link rel=&#34;stylesheet&#34; href=&#34;{{ url_for(&#39;static&#39;, filename=&#39;css/http_cdn.jsdelivr.net_npm_flatpickr_dist_flatpickr.css&#39;) }}&#34;&gt;
    &lt;script src=&#34;{{ url_for(&#39;static&#39;, filename=&#39;js/http_cdn.jsdelivr.net_npm_flatpickr.js&#39;) }}&#34;&gt;&lt;/script&gt;
&lt;/head&gt;
&lt;body&gt;
    &lt;h1&gt;Create Event&lt;/h1&gt;
    &lt;form onsubmit=&#34;combineAndSubmit()&#34; method=&#34;POST&#34;&gt;
        {{ form.hidden_tag() }}&lt;!DOCTYPE html&gt;
&lt;html lang=&#34;en&#34;&gt;
&lt;head&gt;
    &lt;title&gt;Edit Event&lt;/title&gt;
    &lt;link rel=&#34;stylesheet&#34; href=&#34;{{ url_for(&#39;static&#39;, filename=&#39;css/http_cdn.jsdelivr.net_npm_flatpickr_dist_flatpickr.css&#39;) }}&#34;&gt;
    &lt;script src=&#34;{{ url_for(&#39;static&#39;, filename=&#39;js/http_cdn.jsdelivr.net_npm_flatpickr.js&#39;) }}&#34;&gt;&lt;/script&gt;
&lt;/head&gt;
&lt;body&gt;
    &lt;h1&gt;Edit Event&lt;/h1&gt;
    &lt;form onsubmit=&#34;combineAndSubmit()&#34; method=&#34;POST&#34;&gt;
        {{ form.hidden_tag() }}
        &lt;div&gt;
            {{ form.title.label(class=&#34;form-label&#34;) }}
            {{ form.title(class=&#34;form-control&#34;) }}
            {% for error in form.title.errors %}
                &lt;span style=&#34;color: red;&#34;&gt;{{ error }}&lt;/span&gt;
            {% endfor %}
        &lt;/div&gt;
        &lt;div&gt;
            {{ form.start_date.label(class=&#34;form-label&#34;) }}
            {{ form.start_date(class=&#34;form-control flatpickr&#34;,id=&#34;start_date&#34;) }}
            {% for error in form.start_date.errors %}
                &lt;span style=&#34;color: red;&#34;&gt;{{ error }}&lt;/span&gt;
            {% endfor %}
        &lt;/div&gt;
            &lt;div&gt;
            {{ form.event_time.label(class=&#34;form-label&#34;) }}
            {{ form.event_time(class=&#34;form-control flatpickr&#34;,id=&#34;event_time&#34;) }}
            {% for error in form.event_time.errors %}
                &lt;span style=&#34;color: red;&#34;&gt;{{ error }}&lt;/span&gt;
            {% endfor %}
        &lt;/div&gt;
        &lt;div&gt;
            {{ form.end_date.label(class=&#34;form-label&#34;) }}
            {{ form.end_date(class=&#34;form-control, flatpickr&#34;,id=&#34;end_date&#34;) }}
            {% for error in form.end_date.errors %}
                &lt;span style=&#34;color: red;&#34;&gt;{{ error }}&lt;/span&gt;
            {% endfor %}
        &lt;/div&gt;
        &lt;div&gt;
            {{ form.location.label(class=&#34;form-label&#34;) }}
            {{ form.location(class=&#34;form-control&#34;) }}
            {% for error in form.location.errors %}
                &lt;span style=&#34;color: red;&#34;&gt;{{ error }}&lt;/span&gt;
            {% endfor %}
        &lt;/div&gt;
        &lt;div&gt;
            {{ form.description.label(class=&#34;form-label&#34;) }}
            {{ form.description(class=&#34;form-control&#34;) }}
            {% for error in form.description.errors %}
                &lt;span style=&#34;color: red;&#34;&gt;{{ error }}&lt;/span&gt;
            {% endfor %}
        &lt;/div&gt;
        &lt;div&gt;
            &lt;label for=&#34;repeatSelect&#34;&gt;{{ form.repeat.label(class=&#34;form-label&#34;) }}&lt;/label&gt;
            &lt;select name=&#34;repeat&#34; id=&#34;repeatSelect&#34; class=&#34;form-control&#34;&gt;
                &lt;option value=&#34;none&#34; {% if form.repeat.data == &#39;none&#39; %}selected{% endif %}&gt;无&lt;/option&gt;
                &lt;option value=&#34;daily&#34; {% if form.repeat.data == &#39;daily&#39;%}selected{% endif %}&gt;每天&lt;/option&gt;
                &lt;option value=&#34;weekly&#34; {% if form.repeat.data == &#39;weekly&#39; %}selected{% endif %}&gt;每周&lt;/option&gt;
                &lt;option value=&#34;month&#34; {% if form.repeat.data == &#39;month&#39; %}selected{% endif %}&gt;每月&lt;/option&gt;
            &lt;/select&gt;
            {% for error in form.repeat.errors %}
                &lt;span style=&#34;color: red;&#34;&gt;{{ error }}&lt;/span&gt;
            {% endfor %}
        &lt;/div&gt;
        &lt;div&gt;
            &lt;label for=&#34;reminder_type&#34;&gt;{{ form.reminder_type.label(class=&#34;form-label&#34;) }}&lt;/label&gt;
            &lt;select name=&#34;reminder_type&#34; id=&#34;reminder_type&#34; class=&#34;form-control&#34;&gt;
                &lt;option value=&#34;none&#34; {% if form.reminder_type.data == &#39;none&#39; %}selected{% endif %}&gt;无&lt;/option&gt;
                &lt;option value=&#34;email&#34; {% if form.reminder_type.data == &#39;email&#39; %}selected{% endif %}&gt;邮箱&lt;/option&gt;
                &lt;option value=&#34;phone&#34; {% if form.reminder_type.data == &#39;phone&#39; %}selected{% endif %}&gt;手机&lt;/option&gt;
                &lt;option value=&#34;pepper&#34; {% if form.reminder_type.data == &#39;pepper&#39; %}selected{% endif %}&gt;机器人&lt;/option&gt;
            &lt;/select&gt;
            {% for error in form.reminder_type.errors %}
                &lt;span style=&#34;color: red;&#34;&gt;{{ error }}&lt;/span&gt;
            {% endfor %}
        &lt;/div&gt;
        &lt;div&gt;
            {{ form.reminder_time.label(class=&#34;form-label&#34;) }}
            {{ form.reminder_time(class=&#34;form-control&#34;,style=&#34;display: none;&#34;) }}
             &lt;input type=&#34;button&#34; value=&#34;设置提醒&#34; onclick=&#34;showReminderOptions()&#34;&gt;
            {% for error in form.reminder_time.errors %}
                &lt;span style=&#34;color: red;&#34;&gt;{{ error }}&lt;/span&gt;
            {% endfor %}
        &lt;/div&gt;
        &lt;div id=&#34;reminderOptions&#34; style=&#34;display: none;&#34;&gt;
        &lt;label for=&#34;daysInput&#34;&gt;天数：&lt;/label&gt;
        &lt;input type=&#34;number&#34; id=&#34;daysInput&#34; name=&#34;daysInput&#34; min=&#34;0&#34;&gt;
        &lt;br&gt;
        &lt;label for=&#34;hoursInput&#34;&gt;小时：&lt;/label&gt;
        &lt;select id=&#34;hoursInput&#34; name=&#34;hoursInput&#34;&gt;
            &lt;!-- 使用 JavaScript 动态生成选项 --&gt;
        &lt;/select&gt;
        &lt;br&gt;
        &lt;label for=&#34;minutesInput&#34;&gt;分钟：&lt;/label&gt;
        &lt;select id=&#34;minutesInput&#34; name=&#34;minutesInput&#34;&gt;
            &lt;!-- 使用 JavaScript 动态生成选项 --&gt;
        &lt;/select&gt;
        &lt;/div&gt;
        &lt;div&gt;
        &lt;!-- 隐藏的input元素用于存储提醒时间 --&gt;
            &lt;input type=&#34;hidden&#34; name=&#34;reminder_time&#34; id=&#34;reminder_time&#34; value=&#34;&#34;&gt;
        &lt;/div&gt;
        &lt;div&gt;
            {{ form.submit()}}
        &lt;/div&gt;
    &lt;/form&gt;
&lt;script&gt;
    flatpickr(&#39;#start_date&#39;, {
        enableTime: false,
        dateFormat: &#34;Y-m-d&#34;,
    });
    flatpickr(&#39;#event_time&#39;, {
        enableTime: true,
        noCalendar: true, // 禁用日期选择
        time_24hr: true, // 使用24小时制
        dateFormat: &#34;H:i&#34;,
    });
    flatpickr(&#39;#end_date&#39;, {
        enableTime: false,
        dateFormat: &#34;Y-m-d&#34;,
    });
&lt;/script&gt;
&lt;script&gt;
    function populateOptions(element, max) {
        for (let i = 0; i &lt;= max; i&#43;&#43;) {
            const option = document.createElement(&#39;option&#39;);
            option.value = i;
            option.text = i;
            element.appendChild(option);
        }
    }
    // 动态生成小时和分钟的选项
    populateOptions(document.getElementById(&#39;hoursInput&#39;), 24);
    populateOptions(document.getElementById(&#39;minutesInput&#39;), 60);

    var isReminderChanged = false; // 添加标志变量，用于判断是否更改了提醒时间
    function showReminderOptions() {
        // 显示提醒选项时将标志变量设置为 true
        isReminderChanged = true;

        var reminderTime = parseFloat(document.getElementById(&#34;reminder_time&#34;).value);
        // 计算天数、小时和分钟
        var days = Math.floor(reminderTime / (24 * 60));
        var hours = Math.floor((reminderTime % (24 * 60)) / 60);
        var minutes = reminderTime % 60;

        // 更新隐藏的三个选项框的值
        document.getElementById(&#34;daysInput&#34;).value = days;
        document.getElementById(&#34;hoursInput&#34;).value = hours;
        document.getElementById(&#34;minutesInput&#34;).value = minutes;
        // 显示设置提醒时间的选项
        document.getElementById(&#39;reminderOptions&#39;).style.display = &#39;block&#39;;
    }

    function combineAndSubmit() {
        if (isReminderChanged) { // 只有在提醒时间发生更改时才执行以下逻辑
            var daysInput = parseFloat(document.getElementById(&#39;daysInput&#39;).value) || 0;
            var hoursInput = parseFloat(document.getElementById(&#39;hoursInput&#39;).value) || 0;
            var minutesInput = parseFloat(document.getElementById(&#39;minutesInput&#39;).value) || 0;

            // 转换为分钟
            var totalMinutes = daysInput * 24 * 60 &#43; hoursInput * 60 &#43; minutesInput;

            // 将totalMinutes设置到表单的form.reminder_time字段
            document.getElementById(&#39;reminder_time&#39;).value = totalMinutes;

            // 可以将表单提交到服务器或进行其他操作
            alert(&#34;总时间（分钟）：&#34; &#43; totalMinutes);
        }
         // 重置标志变量
        isReminderChanged = false;
    }
&lt;/script&gt;
&lt;/body&gt;
&lt;/html&gt;

        &lt;div&gt;
            {{ form.title.label(class=&#34;form-label&#34;) }}
            {{ form.title(class=&#34;form-control&#34;) }}
            {% for error in form.title.errors %}
                &lt;span style=&#34;color: red;&#34;&gt;{{ error }}&lt;/span&gt;
            {% endfor %}
        &lt;/div&gt;
        &lt;div&gt;
            {{ form.start_date.label(class=&#34;form-label&#34;) }}
            {{ form.start_date(class=&#34;form-control flatpickr&#34;,id=&#34;start_date&#34;) }}
            {% for error in form.start_date.errors %}
                &lt;span style=&#34;color: red;&#34;&gt;{{ error }}&lt;/span&gt;
            {% endfor %}
        &lt;/div&gt;
            &lt;div&gt;
            {{ form.event_time.label(class=&#34;form-label&#34;) }}
            {{ form.event_time(class=&#34;form-control flatpickr&#34;,id=&#34;event_time&#34;) }}
            {% for error in form.event_time.errors %}
                &lt;span style=&#34;color: red;&#34;&gt;{{ error }}&lt;/span&gt;
            {% endfor %}
        &lt;/div&gt;
        &lt;div&gt;
            {{ form.end_date.label(class=&#34;form-label&#34;) }}
            {{ form.end_date(class=&#34;form-control, flatpickr&#34;,id=&#34;end_date&#34;) }}
            {% for error in form.end_date.errors %}
                &lt;span style=&#34;color: red;&#34;&gt;{{ error }}&lt;/span&gt;
            {% endfor %}
        &lt;/div&gt;
        &lt;div&gt;
            {{ form.location.label(class=&#34;form-label&#34;) }}
            {{ form.location(class=&#34;form-control&#34;) }}
            {% for error in form.location.errors %}
                &lt;span style=&#34;color: red;&#34;&gt;{{ error }}&lt;/span&gt;
            {% endfor %}
        &lt;/div&gt;
        &lt;div&gt;
            {{ form.description.label(class=&#34;form-label&#34;) }}
            {{ form.description(class=&#34;form-control&#34;) }}
            {% for error in form.description.errors %}
                &lt;span style=&#34;color: red;&#34;&gt;{{ error }}&lt;/span&gt;
            {% endfor %}
        &lt;/div&gt;
        &lt;div&gt;
            &lt;label for=&#34;repeatSelect&#34;&gt;{{ form.repeat.label(class=&#34;form-label&#34;) }}&lt;/label&gt;
            &lt;select name=&#34;repeat&#34; id=&#34;repeatSelect&#34; class=&#34;form-control&#34;&gt;
                &lt;option value=&#34;none&#34;&gt;无&lt;/option&gt;
                &lt;option value=&#34;daily&#34;&gt;每天&lt;/option&gt;
                &lt;option value=&#34;weekly&#34;&gt;每周&lt;/option&gt;
                &lt;option value=&#34;month&#34;&gt;每月&lt;/option&gt;
            &lt;/select&gt;
            {% for error in form.repeat.errors %}
                &lt;span style=&#34;color: red;&#34;&gt;{{ error }}&lt;/span&gt;
            {% endfor %}
        &lt;/div&gt;
        &lt;div&gt;
            &lt;label for=&#34;reminder_type&#34;&gt;{{ form.reminder_type.label(class=&#34;form-label&#34;) }}&lt;/label&gt;
            &lt;select name=&#34;reminder_type&#34; id=&#34;reminder_type&#34; class=&#34;form-control&#34;&gt;
                &lt;option value=&#34;none&#34;&gt;无&lt;/option&gt;
                &lt;option value=&#34;email&#34;&gt;邮箱&lt;/option&gt;
                &lt;option value=&#34;phone&#34;&gt;手机&lt;/option&gt;
                &lt;option value=&#34;pepper&#34;&gt;机器人&lt;/option&gt;
            &lt;/select&gt;
            {% for error in form.reminder_type.errors %}
                &lt;span style=&#34;color: red;&#34;&gt;{{ error }}&lt;/span&gt;
            {% endfor %}
        &lt;/div&gt;
        &lt;div&gt;
            {{ form.reminder_time.label(class=&#34;form-label&#34;) }}
             &lt;input type=&#34;button&#34; value=&#34;设置提醒&#34; onclick=&#34;showReminderOptions()&#34;&gt;
            {% for error in form.reminder_time.errors %}
                &lt;span style=&#34;color: red;&#34;&gt;{{ error }}&lt;/span&gt;
            {% endfor %}
        &lt;/div&gt;
        &lt;div id=&#34;reminderOptions&#34; style=&#34;display: none;&#34;&gt;
        &lt;label for=&#34;daysInput&#34;&gt;天数：&lt;/label&gt;
        &lt;input type=&#34;number&#34; id=&#34;daysInput&#34; name=&#34;daysInput&#34; min=&#34;0&#34;&gt;
        &lt;br&gt;
        &lt;label for=&#34;hoursInput&#34;&gt;小时：&lt;/label&gt;
        &lt;select id=&#34;hoursInput&#34; name=&#34;hoursInput&#34;&gt;
            &lt;!-- 使用 JavaScript 动态生成选项 --&gt;
        &lt;/select&gt;
        &lt;br&gt;
        &lt;label for=&#34;minutesInput&#34;&gt;分钟：&lt;/label&gt;
        &lt;select id=&#34;minutesInput&#34; name=&#34;minutesInput&#34;&gt;
            &lt;!-- 使用 JavaScript 动态生成选项 --&gt;
        &lt;/select&gt;
        &lt;/div&gt;
        &lt;div&gt;
        &lt;!-- 隐藏的input元素用于存储提醒时间 --&gt;
            &lt;input type=&#34;hidden&#34; name=&#34;reminder_time&#34; id=&#34;reminder_time&#34; value=&#34;&#34;&gt;
        &lt;/div&gt;
        &lt;div&gt;
            {{ form.submit()}}
        &lt;/div&gt;
    &lt;/form&gt;
&lt;script&gt;
    flatpickr(&#39;#start_date&#39;, {
        enableTime: false,
        dateFormat: &#34;Y-m-d&#34;,
    });
    flatpickr(&#39;#event_time&#39;, {
        enableTime: true,
        noCalendar: true, // 禁用日期选择
        time_24hr: true, // 使用24小时制
        dateFormat: &#34;H:i&#34;,
    });
    flatpickr(&#39;#end_date&#39;, {
        enableTime: false,
        dateFormat: &#34;Y-m-d&#34;,
    });
&lt;/script&gt;
&lt;script&gt;
    function populateOptions(element, max) {
        for (let i = 0; i &lt;= max; i&#43;&#43;) {
            const option = document.createElement(&#39;option&#39;);
            option.value = i;
            option.text = i;
            element.appendChild(option);
        }
    }

    // 动态生成小时和分钟的选项
    populateOptions(document.getElementById(&#39;hoursInput&#39;), 24);
    populateOptions(document.getElementById(&#39;minutesInput&#39;), 60);

    function showReminderOptions() {
        // 显示设置提醒时间的选项
        document.getElementById(&#39;reminderOptions&#39;).style.display = &#39;block&#39;;
    }

    function combineAndSubmit() {
        var daysInput = parseFloat(document.getElementById(&#39;daysInput&#39;).value) || 0;
        var hoursInput = parseFloat(document.getElementById(&#39;hoursInput&#39;).value) || 0;
        var minutesInput = parseFloat(document.getElementById(&#39;minutesInput&#39;).value) || 0;

        // 转换为分钟
        var totalMinutes = daysInput * 24 * 60 &#43; hoursInput * 60 &#43; minutesInput;

        // 将totalMinutes设置到表单的form.reminder_time字段
        document.getElementById(&#39;reminder_time&#39;).value = totalMinutes;

        // 可以将表单提交到服务器或进行其他操作
        alert(&#34;总时间（分钟）：&#34; &#43; totalMinutes);
    }
&lt;/script&gt;
&lt;/body&gt;
&lt;/html&gt;

```

# 效果呈现

**注：以下效果图的html代码并非以上的代码，而是通过优化后的效果**

## /event/event_list

![image.png](https://img-1313049298.cos.ap-shanghai.myqcloud.com/note-img/202503071826519.png)


## /event/creat_event

![image.png](https://img-1313049298.cos.ap-shanghai.myqcloud.com/note-img/202503071826520.png)

## /event/edit_event/

![image.png](https://img-1313049298.cos.ap-shanghai.myqcloud.com/note-img/202503071826521.png)


---

> 作者:   
> URL: https://lucazou.github.io/posts/20250307182650/  

