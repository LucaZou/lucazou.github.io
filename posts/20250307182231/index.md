# 【Pepper机器人管家（一）】项目构建

&gt;基于软银的pepper机器人，使用可视化低代码SDK软件choregraphe和web开发工具链开发一个机器人管家。

&lt;!--more--&gt;
# 项目概述

基于软银的pepper机器人，使用可视化低代码SDK软件choregraphe和web开发工具链开发一个机器人管家。考虑到的功能有提醒按时用药，健康疑难解答，康复训练教程，闹钟，跌倒检测功能(内置电话功能与邮件功能)，听歌，读报，跳舞……。

# 项目架构

目前考虑的的功能的实现主要分成两部分，一个是基于choregraphe的机器人模块的开发，一个是基于web开发的网站。将这两部分结合后构成一个机器人管家。
![image.png](https://img-1313049298.cos.ap-shanghai.myqcloud.com/note-img/202503071821176.png)

## Choregraphe

**choregraph主要承担系统的主控并实现提醒、跳舞、交流等功能**
choregraphe是一个低代码的Pepper SDK，内置很多可拖拽的功能盒子，能完成项目的绝大部分需求。其中还有python script盒子能实现一些简单的python脚本，可惜python的版本只有python 2.7，而且很多库没有内置，在是开发的一大困难之一。

![image.png](https://img-1313049298.cos.ap-shanghai.myqcloud.com/note-img/202503071821177.png)

第二大难点是代码的调试和程序的稳定性很差，低代码的坏处就是自由度太低，而且网上的教程太少，整个项目开发只需要一份官方文档，官方文档虽然很多地方不齐全但也是比较完整。[官方文档](http://doc.aldebaran.com/2-4/index_dev_guide.html)

## Web开发

**Web模块主要承担日程的设置、信息的推送、用户管理等功能**
web开发选用的技术栈为：flask&#43;MySQL&#43;jinja2的前后端不分离开发流程，页面使用原生js&#43;html&#43;css。没有使用前端框架并使用前后端不分离主要是机器人的平板内置的浏览器版本太低，无法兼容大部分的“重型”js框架，将页面资源在后端渲染或许是比较好的方法。工具链使用的是Pycharm&#43;navicat&#43;vscode，部署使用了nginx&#43;gunicorn，部署服务器为阿里云的轻量应用服务器2核4G：Ubuntu。

### **后端：**

后端使用了MVC架构，用蓝图管理路由，并且创建了一些api供机器人调用，机器人与服务器主要同这些api进行通讯。

![image.png](https://img-1313049298.cos.ap-shanghai.myqcloud.com/note-img/202503071821178.png)

#### **数据库**


数据库的搭建使用了MySQL&#43;flask-SQLAlchemy的orm框架。数据库涉及到用户的的注册和登录、用户的的日程CRUD、用户邮件的历史的保存和删除。

![image.png](https://img-1313049298.cos.ap-shanghai.myqcloud.com/note-img/202503071821179.png)

### **前端:**

前端是二次开发的原生js的页面


**主页:**
![image.png](https://img-1313049298.cos.ap-shanghai.myqcloud.com/note-img/202503071821180.png)

**日程管理:**
![image.png](https://img-1313049298.cos.ap-shanghai.myqcloud.com/note-img/202503071821181.png)


完整的项目代码参考GitHub: https://github.com/Lucawell/flaskSchedule-pepper-MVC

---

> 作者: LucaZou  
> URL: https://lucazou.github.io/posts/20250307182231/  

