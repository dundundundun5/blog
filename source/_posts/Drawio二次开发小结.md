---
title: Drawio二次开发小结
date: 2025-12-16 11:31:04
categories:
- 开发
tags: 
- 二次开发
- JavaScript
   

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
-  `git clone https://github.com/jgraph/drawio` 
   - 切换到（签出）标签v24.3.1，这个版本也比较新了，所以我用这个版本，因为最终肯定是大多数功能都用不到的，既然是瘦身那就不需要太新的版本
   - 因为是屎山代码，所以越新的版本防御性编程越强，改动难度越大。
   - 把src/main下面的webapp这个文件夹提取出来，这是浏览器端的所有文件了
- nodejs装好，当前目录下
  - 切换淘宝镜像 `npm config set registry https://registry.npmmirror.com`
  - 安装必要的包（serve用于开启服务） `npm install serve`
  - - npm的dev脚本写成
    
    ```json
    "scripts": {
        "dev": "serve -p 11451"
    }
    ```
  - 运行 `npm run dev` 或者 `serve -p 11451`
    - `11451` 是我设置的端口 ，因为逸一时，误一世

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


### 二次开发重要改动点

- `js/bootstrap.js`

    找到开头代码块
    
    ```javascript
      var urlParams = (function()
      {
          //代码
      })();  
    ```
    在如图位置![](bootstrap1.png)加上代码块
    ```javascript
      params = [
        'dev=1',
        'test=1'
      ]
    ```

    注释掉如图代码块![](bootstrap2.png)

    再注释调如图代码块，这个是仓库源码一旦修改，就会提示校验和不一致![](bootstrap3.png)
   
    最后在如图位置![](bootstrap4.png)加上代码

    ```javascript
        if (location.hostname === 'localhost' || location.hostname === '127.0.0.1') {
            drawDevUrl = `http://${location.host}/`
            geBasePath = `http://${location.host}/js/grapheditor`;
            mxBasePath = `http://${location.host}/mxgraph`
            mxForceIncludes = true
        }
        drawDevUrl = `http://${location.host}/`
        geBasePath = `http://${location.host}/js/grapheditor`;
        mxBasePath = `http://${location.host}/mxgraph`
        mxForceIncludes = true
    ```


    如何加入本地自己定义的js模块？用他的 `mxscript()` ，这样全局都能用，用var声明函数或变量即可，如图所示![](bootstrap5.png)
- `js/PreConfig.js`

   在如图位置![](preconfig1.png)加入如下

   ```javascript
      urlParams.lang = 'zh'; // 国际化默认中文
      // 浏览器模式
      urlParams['browser'] = 1;
      urlParams['gapi'] = 0;
      urlParams['db'] = 0;
      urlParams['od'] = 0;
      urlParams['tr'] = 0;
      urlParams['gh'] =0;
      urlParams['gl'] =0;
      urlParams['mode'] = 'browser' 
   ```
- `js/diagramly/App.js`

    （可选）注释掉如图，主要是删掉页面右上方的Share按钮![](diagramly-app1.png)

    （可选）修改
    
    ```javascript
      this.appIcon.style.backgroundColor = '#FFFFFF'
    ```
  
    如图，主要是把左上角的logo的背景色从橙色改为白色，然后禁用了logo的事件监听![](diagramly-app2.png)
- `js/diagramly/Editor.js`

  （可选）在如图位置修改左上角的logo，把图片转成base64字符串就行![](diagramly-editor1.png)

  （可选）在如图位置注释掉代码，这样可以防止默认字体是Helvetica，但是修改字体不在这个文件![](diagramly-editor2.png)
- `js/diagramly/EditorUi.js`

  （可选）加入一个函数，用于在右边样式栏增加一个处理字符串的文本框（它默认的文本框只能处理数字）

  ```javascript
  EditorUi.prototype.addInputDun = function(div, label, defaultName)
  {
      var lbl = document.createElement('label');
      mxUtils.write(lbl, label);
      lbl.setAttribute('for', id);
      div.appendChild(lbl);


      var cb = document.createElement('input');
      cb.style.marginRight = '8px';
      cb.style.marginTop = '16px';
      cb.style.width = '180px';
      cb.setAttribute('type', 'text');
      cb.value = defaultName;
      var id = 'geTextBox-' + Editor.guid();
      cb.id = id;



      div.appendChild(cb);


      mxUtils.br(div);

      return cb;
  };
   ```

   (可选) 在如图位置代码替换为
   ```javascript
   cell.style += ';sketch=1;' + (cell.style.indexOf('fontFamily=') == -1 || cell.style.indexOf('fontFamily=Times New Roman;') > -1?
											'fontFamily=Architects Daughter;fontSource=https%3A%2F%2Ffonts.googleapis.com%2Fcss%3Ffamily%3DArchitects%2BDaughter;' : '');
   ```

   ，主要是把字体换成Times New Roman![](diagramly-editorui1.png)
- `js/diagramly/Menus.js`

    会优先调用 `js/grapheditor/Menus.js` 的`init`逻辑，随后再调用本文件的`init`逻辑

    （可选）比如说，我把导出-导出为html的逻辑改成了导出-导出为excel的逻辑，那么改动就如图所示![](diagramly-menus1.png)

    逻辑变了但是选项文字没变，那么要把导出html改成导出excel![](diagramly-menus2.png)
    
    ```javascript
    this.addMenuItems(menu, ['exportExcel', 'exportXml'], parent);
    ```
    
    （可选）最后还可以注释掉一部分 `addMenusItems` 和 `addItem` 改造菜单栏，我这边就把分享到google drive这块去掉了![](diagramly-menus3.png)
    