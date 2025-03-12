# Flask MVC Pepper Schedule 项目文档

&gt;

&lt;!--more--&gt;
# **Flask MVC Pepper Schedule 项目文档**
## **1. 项目概述**
- **项目名称**: Flask MVC Pepper Schedule
- **版本**: 1.0（根据代码推测，无明确版本号）
- **描述**: 一个基于 Flask 的 Web 应用程序，用于用户注册登录、日程管理、邮件与短信提醒、文件上传及管理员后台管理。项目采用 MVC 架构，通过 Blueprint 模块化设计，支持 RESTful API 和前端模板渲染。
- **目标用户**: 需要日程管理和提醒功能的用户，以及需要管理用户和事件的管理员。
- **当前日期**: 2025年3月6日（基于您的输入）

---

## **2. 功能模块**

### **2.1 用户管理**
- **功能**:
  - **注册**: 用户通过邮箱、用户名、电话和密码注册账户。
  - **登录**: 使用邮箱和密码登录，支持“记住我”功能。
  - **登出**: 注销当前用户会话。
  - **个人信息**: 查看用户名、邮箱和电话。
- **路由**:
  - `/register` (GET/POST): 注册页面
  - `/login` (GET/POST): 登录页面
  - `/logout` (GET): 登出
  - `/profile` (GET/POST): 个人信息页面
- **依赖**: `Flask-Login`, `Flask-WTF`, `werkzeug.security`

### **2.2 事件管理**
- **功能**:
  - **创建事件**: 输入标题、时间、地点、描述、重复类型和提醒设置。
  - **查看事件**: 显示今天事件和过期事件，支持重复事件动态生成。
  - **编辑/删除事件**: 修改或删除已有事件，仅限创建者操作。
  - **日历视图**: 显示所有事件。
  - **RESTful API**: 获取用户事件数据。
- **路由**:
  - `/event_list` (GET): 事件列表
  - `/create_event` (GET/POST): 创建事件
  - `/delete_event/&lt;int:event_id&gt;` (POST): 删除事件
  - `/edit_event/&lt;int:event_id&gt;` (GET/POST): 编辑事件
  - `/calendar` (GET): 日历视图
- **API**:
  - `GET /event/&lt;user_id&gt;`: 获取用户事件
- **依赖**: `Flask-SQLAlchemy`, `Flask-WTF`

### **2.3 提醒功能**
- **功能**:
  - **邮件提醒**: 发送 HTML 格式的提醒邮件，记录发送历史。
  - **短信提醒**: 使用阿里云 SMS 服务发送短信。
  - **浏览器提醒**: 定时检查事件并打印提醒日志（未实现前端通知）。
- **路由**:
  - `/send_email` (GET/POST): 发送邮件
  - `/email_history` (GET): 查看邮件历史
  - `/delete_email_history/&lt;int:email_history_id&gt;` (POST): 删除邮件历史
- **API**:
  - `POST /email`: 发送邮件
  - `POST /sms`: 发送短信
- **依赖**: `Flask-Mail`, `Flask-Scheduler`, `aliyunsdkdysmsapi`

### **2.4 文件上传与相册**
- **功能**:
  - **上传文件**: 支持图片上传，文件按时间戳命名。
  - **相册**: 显示上传的图片列表。
- **路由**:
  - `/album_list` (GET): 相册页面
- **API**:
  - `POST /upload`: 上传文件
- **依赖**: `Flask-Uploads`

### **2.5 管理后台**
- **功能**:
  - 管理 `User`, `Event`, `EmailHistory` 表数据，仅限管理员访问。
  - 支持搜索、排序和关联字段显示。
- **依赖**: `Flask-Admin`

### **2.6 首页与关于**
- **功能**:
  - **首页**: 显示用户的事件列表。
  - **关于**: 静态页面。
- **路由**:
  - `/` (GET): 首页
  - `/about` (GET): 关于页面

---

## **3. 技术栈**
- **后端框架**: Flask
- **数据库**: MySQL (via SQLAlchemy)
- **用户认证**: Flask-Login
- **表单验证**: Flask-WTF
- **邮件服务**: Flask-Mail (QQ SMTP)
- **文件上传**: Flask-Uploads
- **定时任务**: Flask-Scheduler
- **管理后台**: Flask-Admin
- **RESTful API**: Flask-RESTful
- **短信服务**: 阿里云 SMS SDK
- **日志**: Python `logging`
- **环境变量**: `python-dotenv`

---

## **4. 安装与运行指南**

### **4.1 环境要求**
- Python 3.6&#43;
- MySQL 数据库
- 阿里云 SMS 服务账号（可选）
- QQ 邮箱 SMTP 配置

### **4.2 安装步骤**
1. **克隆项目**:
   ```bash
   git clone &lt;repository_url&gt;
   cd flask-mvc-pepper-schedule
   ```
2. **创建虚拟环境**:
   ```bash
   python -m venv venv
   source venv/bin/activate  # Linux/Mac
   venv\Scripts\activate     # Windows
   ```
3. **安装依赖**:
   ```bash
   pip install -r requirements.txt
   ```
   *注*: 项目未提供 `requirements.txt`，建议手动安装以下库：
   ```
   flask flask-sqlalchemy flask-login flask-mail flask-uploads flask-admin flask-restful flask-wtf flask-scheduler pymysql python-dotenv aliyunsdkcore aliyunsdkdysmsapi werkzeug
   ```
4. **配置环境变量**:
   - 创建 `.env` 文件，添加以下内容：
     ```
     MAIL_PASSWORD=&lt;your_qq_email_password&gt;
     ALIYUN_ACCESS_KEY_ID=&lt;your_aliyun_access_key_id&gt;
     ALIYUN_ACCESS_KEY_SECRET=&lt;your_aliyun_access_key_secret&gt;
     API_TOKEN=&lt;your_api_token&gt;
     ```
5. **初始化数据库**:
   - 配置 `SQLALCHEMY_DATABASE_URI`（默认 `mysql&#43;pymysql://root:123456@localhost/schedule`）
   - 运行以下命令创建表：
     ```python
     from app import db, create_app
     app = create_app()
     with app.app_context():
         db.create_all()
     ```
6. **运行应用**:
   ```bash
   python main.py
   ```
   - 默认访问: `http://127.0.0.1:5000`

---

## **5. API 说明**

### **5.1 事件资源 (`EventResource`)**
- **GET /event/&lt;user_id&gt;**:
  - **描述**: 获取指定用户的事件。
  - **参数**: `user_id` (int)
  - **Header**: `Authorization: &lt;API_TOKEN&gt;`
  - **返回**: JSON
    ```json
    {
      &#34;events&#34;: [{&#34;id&#34;: 1, &#34;title&#34;: &#34;Meeting&#34;, ...}],
      &#34;expired_events&#34;: [{&#34;id&#34;: 2, &#34;title&#34;: &#34;Old Event&#34;, ...}]
    }
    ```

### **5.2 消息资源 (`MessageResource`)**
- **POST /message**:
  - **描述**: 接收消息并存储。
  - **Body**: `{&#34;message&#34;: &#34;test&#34;}`
  - **返回**: JSON
    ```json
    {&#34;message&#34;: &#34;test&#34;}
    ```

### **5.3 邮件资源 (`EmailResource`)**
- **POST /email**:
  - **描述**: 发送邮件。
  - **Header**: `Authorization: &lt;API_TOKEN&gt;`
  - **Body**: `{&#34;title&#34;: &#34;Test&#34;, &#34;content&#34;: &#34;Hello&#34;, &#34;recipients&#34;: [&#34;user@example.com&#34;]}`
  - **返回**: JSON
    ```json
    {&#34;message&#34;: &#34;邮件发送成功&#34;}
    ```

### **5.4 短信资源 (`SMSResource`)**
- **POST /sms**:
  - **描述**: 发送短信。
  - **Header**: `Authorization: &lt;API_TOKEN&gt;`
  - **Body**: `{&#34;phone_number&#34;: &#34;12345678901&#34;, &#34;message&#34;: &#34;Hello&#34;}`
  - **返回**: JSON
    ```json
    {&#34;status&#34;: &#34;success&#34;, &#34;result&#34;: {...}}
    ```

### **5.5 文件上传资源 (`FileUploadResource`)**
- **POST /upload**:
  - **描述**: 上传图片文件。
  - **Body**: Form-data (`photo`: 文件)
  - **返回**: JSON
    ```json
    {&#34;status&#34;: &#34;success&#34;, &#34;file_url&#34;: &#34;/static/uploads/images/2025/03/06/1234567890.jpg&#34;}
    ```

---

## **6. 数据库设计**

### **6.1 User 表**
- **表名**: `users`
- **字段**:
  - `id`: Integer, 主键
  - `name`: String(255), 非空
  - `email`: String(255), 非空，唯一
  - `phone`: String(255), 非空
  - `password_hash`: String(255), 非空
  - `is_admin`: Boolean, 默认 False
- **关系**: 一对多 (`Event`, `EmailHistory`)

### **6.2 Event 表**
- **表名**: `events`
- **字段**:
  - `id`: Integer, 主键
  - `user_id`: Integer, 外键 (`users.id`)
  - `title`: String(255), 非空
  - `start_date`: Date, 非空
  - `event_time`: Time
  - `end_date`: Date
  - `location`: String(255)
  - `description`: String(255)
  - `repeat`: String(255), 默认 &#34;none&#34;
  - `reminder_type`: String(255), 默认 &#34;none&#34;
  - `reminder_time`: Float, 默认 0
- **关系**: 多对一 (`User`)

### **6.3 EmailHistory 表**
- **表名**: `email_history`
- **字段**:
  - `id`: Integer, 主键
  - `user_id`: Integer, 外键 (`users.id`)
  - `title`: String(255), 非空
  - `content`: Text, 非空
  - `recipients`: String(255), 非空
  - `sent_at`: DateTime, 默认当前时间
- **关系**: 多对一 (`User`)

---

## **7. 潜在改进建议**

### **7.1 功能完善**
- 添加密码重置和用户个人信息编辑功能。
- 增强重复事件逻辑，支持更复杂的规则（如“每月的第几天”）。
- 实现浏览器提醒的前端通知（使用 WebSocket 或推送服务）。

### **7.2 性能优化**
- RESTful API 添加分页和缓存机制。
- 使用分布式任务队列（如 Celery）优化定时任务。

### **7.3 用户体验**
- 添加事件搜索和过滤功能。
- 相册支持分类、删除和预览。

### **7.4 安全性**
- 限制上传文件的大小和类型。
- API 添加速率限制和详细错误码。

### **7.5 部署**
- 配置生产环境（如 Gunicorn &#43; Nginx）。
- 生成 API 文档（如 Swagger）。




---

> 作者:   
> URL: https://lucazou.github.io/posts/20250307182802/  

