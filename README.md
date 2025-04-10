## 对hexo-helper-live2d默认模型的优化

谁不想给自己的博客界面添加一个可爱的看板娘呢？来看看如何实现吧。

<img src="https://www.sanye.blog/images/%E6%90%AD%E5%BB%BAhexo/live2D.png" style="zoom: 67%;">

在github上有个star非常多的，基于hexo的live2D项目：[EYHN/hexo-helper-live2d](https://github.com/EYHN/hexo-helper-live2d)

部署起来也非常方便。

```bash
npm install --save hexo-helper-live2d
```

安装完成之后再在`_config.yml`文件下添加如下配置

```yaml
live2d:
  enable: true
  scriptFrom: local
  pluginRootPath: live2dw/
  pluginJsPath: lib/
  pluginModelPath: assets/
  tagMode: false
  log: false
  #注释掉mode，使用的就是默认模型
  #mode:
  	#use:
  display:
    position: left
    width: 100
    height: 200
  mobile:
    show: true
  react:
    opacity: 0.7
```

其实，上述项目`hexo-helper-live2d`主要是由项目[live2d-widget.js](https://github.com/xiazeyu/live2d-widget.js)开发而来，方便其迁移到hexo。特别是上述看板娘模型，其实就是`live2d-widget.js`的默认模型。我们下载`hexo-helper-live2d`同时也会下载`live2d-widget.js`，删除`hexo-helper-live2d`也会删除`live2d-widget.js`，`hexo-helper-live2d`依赖的主要是`live2d-widget.js`项目生产打包后的代码，即`lib`文件下的js文件。

由于我自己的博客一直都用的是上述看板娘(`shizuku`)，因为这个模型好看交互性还好，用久了发现还是存在一些问题的，在网上也一直找不到解决方法，所以只能自己扒源码了😭，下面是优化的地方（修改的是`live2d-widget.js`项目的源码）：

* 我把这个模型所有相关资料下载到了本地，最终部署到vercel，这样就不需要借助cdn服务了，理论上是能提高加载速度的，模型所需文件都放到仓库中了。

* 我还注意到点击模型的头部，只会加载动画不会触发声音，于是我查看了源码发现：

  ```js
  // src\cDefine.js
  
  MOTION_GROUP_IDLE : "idle",
  MOTION_GROUP_TAP_BODY : "tap_body",
  MOTION_GROUP_FLICK_HEAD : "flick_head", // unused
  MOTION_GROUP_PINCH_IN : "pinch_in", // unused
  MOTION_GROUP_PINCH_OUT : "pinch_out", // unused
  MOTION_GROUP_SHAKE : "shake", // unused
  ```

  源码确实未启用部分功能，但只需要修改`src\cManager.js`文件：

  ```js
  // src\cManager.js
  cManager.prototype.tapEvent = function (x, y) {
    //...
    for (var i = 0; i < this.models.length; i++) {
      if (this.models[i].hitTest(cDefine.HIT_AREA_HEAD, x, y)) {
        this.eventemitter.emit('tapface');
        if (cDefine.DEBUG_LOG)
          console.log('Tap face.');
        this.models[i].startRandomMotion(cDefine.MOTION_GROUP_FLICK_HEAD,
          cDefine.PRIORITY_NORMAL);//这段代码模仿下面的代码是修改后的，还真有用
      }
      else if (this.models[i].hitTest(cDefine.HIT_AREA_BODY, x, y)) {
        this.eventemitter.emit('tapbody');
        if (cDefine.DEBUG_LOG)
          console.log('Tap body.' + ' models[' + i + ']');
  
        this.models[i].startRandomMotion(cDefine.MOTION_GROUP_TAP_BODY,
          cDefine.PRIORITY_NORMAL);
      }
    }
    return true;
  };
  ```

* 手动下载模型资源的时候，我发现，模型其实有很多动作和语音，但是之前我从未触发过，因为那些动作和语音的触发方式比较特殊，在PC端无法触发，于是我把其他`motion`和对应的`audio`，都放到了`tap_body`配置属性，这样点击身体就能触发更多动作和语音了。

* 还有一个问题，每次点击总是触发重复的动作，而不是随机触发动作，肯定是代码出了问题，于是我扒源代码扒了很久，尝试性修改了许多次，失败了很多次，终于找到了解决方法：

  ```js
  // src\cModel.js
  cModel.prototype.setFadeInFadeOut = function(name, no, priority, motion)
  {
      var motionName = this.modelSetting.getMotionFile(name, no);
      motion.setFadeIn(this.modelSetting.getMotionFadeIn(name, no));
      motion.setFadeOut(this.modelSetting.getMotionFadeOut(name, no));
      if (cDefine.DEBUG_LOG)
              console.log("Start motion : " + motionName);
      if (this.modelSetting.getMotionSound(name, no) == null)
      {
          this.mainMotionManager.startMotionPrio(motion, priority);
      }
      else
      {
          var soundName = this.modelSetting.getMotionSound(name, no);
          // var player = new Sound(this.modelHomeDir + soundName);
          var snd = document.createElement("audio");
          snd.src = this.modelHomeDir + soundName;
          snd.play();
          if (cDefine.DEBUG_LOG)
              console.log("Start sound : " + soundName);
          //下面的代码是新加的，作用是每次点击都加载新的motion（动作）
          this.loadMotion(name, this.modelHomeDir + motionName, (mtn)=> {
              motion = mtn;
              this.mainMotionManager.startMotionPrio(motion, priority);
          });    
      }
  }
  ```

* 还还还有一个问题，就是语音和角色动画不同步的问题，语音总比动画要慢，这一点，可以通过把模型`motion`和`audio`资源下载到本地来缓解，这一点我们已经实现了，为了进一步解决问题，我决定提前加载所有`audio`，还是修改`cModel.js`文件：

  ```js
  // src\cModel.js
  export function cModel()
  {
      //L2DBaseModel.apply(this, arguments);
      L2DBaseModel.prototype.constructor.call(this);
  
      this.modelHomeDir = "";
      this.modelSetting = null;
      this.tmpMatrix = [];
  +   this.audioObj = {}
  }
  ```

  ```js
  // src\cModel.js
  cModel.prototype.preloadMP3 = function(url) {
      var xhr = new XMLHttpRequest();
      xhr.open('GET', url, true); // 第三个参数为 true 表示异步请求
      xhr.responseType = 'arraybuffer'; // 设置响应类型为二进制数据
      xhr.onload =  () => {
          if (xhr.status === 200) {
              const arrayBuffer = xhr.response
              // 将 URL 和对应的 Base64 数据存储到 audioObj 中
              this.audioObj[url] = arrayBuffer
          } else {
              console.error('Failed to preload MP3 file. Status:', xhr.status);
          }
      };
      xhr.onerror = function () {
          console.error('An error occurred while preloading the MP3 file.');
      };
      xhr.send();
  }
  cModel.prototype.preloadMP3Group = function(name){
      for (var i = 0; i < this.modelSetting.getMotionNum(name); i++)
          {
              let audio = this.modelSetting.getMotionSound(name, i);
              this.preloadMP3(this.modelHomeDir + audio)
          }
  }
  ```

  ```js
  // src\cModel.js
  cModel.prototype.load = function(gl, modelSettingPath, callback)
  {
       //其他代码
       thisRef.live2DModel.saveParam();
  
       thisRef.preloadMotionGroup(cDefine.MOTION_GROUP_IDLE);
  +    thisRef.preloadMP3Group(cDefine.MOTION_GROUP_FLICK_HEAD)
  +    thisRef.preloadMP3Group(cDefine.MOTION_GROUP_TAP_BODY)
       thisRef.mainMotionManager.stopAllMotions();
  
       thisRef.setUpdating(false);
       thisRef.setInitialized(true);
       //其他代码
  }
  ```

  ```js
  // src\cModel.js
  cModel.prototype.setFadeInFadeOut = function(name, no, priority, motion)
  {
      var motionName = this.modelSetting.getMotionFile(name, no);
  
      motion.setFadeIn(this.modelSetting.getMotionFadeIn(name, no));
      motion.setFadeOut(this.modelSetting.getMotionFadeOut(name, no));
  
      if (cDefine.DEBUG_LOG)
              console.log("Start motion : " + motionName);
  
      if (this.modelSetting.getMotionSound(name, no) == null)
      {
          this.mainMotionManager.startMotionPrio(motion, priority);
      }
      else
      {
          var soundName = this.modelSetting.getMotionSound(name, no);
          
          var snd = document.createElement("audio");
          // snd.src = this.modelHomeDir + soundName;
  +       const blob = new Blob([this.audioObj[this.modelHomeDir+soundName]], { type: 'audio/mpeg' });
  +       const objectURL = URL.createObjectURL(blob);
  +       snd.src = objectURL
          snd.play();
          if (cDefine.DEBUG_LOG)
              console.log("Start sound : " + soundName);
  
          this.loadMotion(name, this.modelHomeDir + motionName, (mtn) =>{
              motion = mtn;
              this.mainMotionManager.startMotionPrio(motion, priority);
          });
      }
  }
  ```

* 然后重新对项目进行生产打包即可，需要先下载所需包

  ```bash
  npm i
  ```

  然后修改一下`package.json`文件

  ```json
  "scripts": {
      "build:prod": "npx webpack --env prod --progress --colors && npm run build:module",
      "build:module": "npx webpack --env prod --progress --colors --config webpack.config.common.js",
  }
  ```

  再运行：

  ```bash
  npm run build:prod
  ```

  即可。

  

其他可用模型：[xiazeyu/live2d-widget-models: Model library for live2d-widget.js](https://github.com/xiazeyu/live2d-widget-models)（交互功能不如默认模型）

其他live2D项目：[stevenjoezhang/live2d-widget: 把萌萌哒的看板娘抱回家 (ノ≧∇≦)ノ | Live2D widget for web platform](https://github.com/stevenjoezhang/live2d-widget)