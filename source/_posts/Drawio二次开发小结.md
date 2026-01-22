---
title: Drawio二次开发小结
date: 2025-12-16 11:31:04
tags:
---
## 需求
- 入职单位有一个画图软件的需求，B/S 端比较方便，每当更新刷新下网页行
- 业务图由至多20种业务相关的多边形、组合图形、线段构成
- 业务图导出为Excel，Excel的导出结果有模板
## 须知

### 关于Drawio
- Drawio是一个画图软件，目前能用到的是Drawio-Desktop，要用到网页版的只能上他们官网
  - 开源，能提issue
  - 不接受任何陌生人，仅限团队内成员PR
- 是electron + web三件套的模式，渲染进程内的所有页面和样式都是用纯JavaScript动态构建
  - 所有事件的监听、回调都是手动重写，一点没用到原生的事件系统。每个模块的每个事件都重写了，因此代码非常屎山。
  - electron + web三件套组成桌面端 对应仓库 `https://github.com/jgraph/drawio-desktop`
  - web三件套组成web端 对应仓库 `https://github.com/jgraph/drawio` 渲染进程在src/main的webapp
  - 所有图形的外形、交互、事件监听 对应仓库 `https://github.com/jgraph/mxgraph`
- web端自带左上角菜单栏提供基于blob的导入导出，通过浏览器上传或下载文件；桌面端重写了菜单栏变成基于electron主进程的系统Api调用，
### 开发环境准备
-  `git clone https://github.com/jgraph/drawio-desktop.git`
  - 把src/main下面的webapp这个文件夹提取出来，这是浏览器端的所有文件了
- nodejs装好，当前目录下
  - 切换淘宝镜像 `npm config set registry https://registry.npmmirror.com`
  - 安装必要的包（serve用于开启服务） `npm install serve`
  - npm的dev脚本写成
    
    ```json
    "scripts": {
        "dev": "serve -p 11451"
    }
    ```
  - 运行 `npm run dev` 或者 `serve -p 11451`
    - 端口我设置的是 `11451`，因为逸一时，误一世

### 代码结构简析
- 图形库
  - `mxgraph/*` 是图形本体的核心库，也是jgraph原创的图形外观、交互、事件监听库
- 首页
  - `index.html` 全场唯一的首页
  - `js/bootstrap.js` 启动配置，在此处开启dev模式，在此处使用`mxscript()`函数静态加入一些js包（例如xlsx-js-style）
  - `js/PreConfig.js` 启动配置的预设置，可以在此处锁定中文语言
  - `js/diagramly/App.js` 应用启动类，在此处可以更换左上角logo，右上角的共享按钮
- 侧边栏
  - 先执行 `js/grapheditor/Sidebar.js` 侧边栏的每一栏的每个图形的创建细节
  - 再执行 `js/diagramly/Sidebar.js` 侧边栏要包含哪些栏，每一栏的名称，每一栏要包含哪些元素
- 菜单
    - 先执行 `js/grapheditor/Menus.js` 菜单要包含哪些根选项，菜单的样式
    - 再执行 `js/diagramly/Menus.js` 菜单根选项有哪些子选项，每个子选项的实现逻辑
- 样式
  - `js/grapheditor/Format.js` 一口气把右边侧边栏的画布样式（文本、样式），图形样式（文本、样式、调整图形）全部定义掉了，所有js逻辑、所有html和css都在这个文件里用js写死。
- 画布
  - `js/diagramly/Editor.js` 包含了所有画布的快捷键逻辑和各种图形变化的事件监听，屎中屎
  - `js/diagramly/EditorUi.js` 包含了所有画布的导出导入的各种额外窗口的图形化定义
