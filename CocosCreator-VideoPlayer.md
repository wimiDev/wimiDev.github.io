# 关于VideoPlayer的爬坑记录
使用的版本的CocosCreator v1.9.3，运行环境是在微信浏览器上，我在实现游戏内播放视频中遇到的一些问题，比如视频无法播放，各平台的表现上差异很大。

# [VideoPlayer组件](https://docs.cocos.com/creator/1.9/manual/zh/components/videoplayer.html)
主要用于h5小游戏的视频播放（有点鸡肋），应该是对video组件进行了封装。对于不同的平台有不同的表现，差异性也比较大。

# 服法用量
用起来还是很简单的，拖进场景里就可以用了。具体使用方法可以参照[CocosCreator](https://docs.cocos.com/creator/1.9/manual/zh/components/videoplayer.html)的官方文档。

# 一些爬过的坑
* VideoPlayer是始终浮在Canvas的上方的。VideoPlayer直接使用网页的video标签来实现的，这个可以通过打开浏览器的开发者模式来查看，他的层级关系。
* 在各平台的表现不一致。比如在安卓的微信浏览器上是默认全屏弹窗显示播放器，在ios的微信浏览器上则只是全屏 不 弹窗显示。如果要更改不同的显示方式，可以修改video标签（这里指的是在网页中的标签，不是creator编辑器里的属性），打个比方，我要设置成Canvas盖住Videoplayer，并且全屏不弹窗。就需要修改两部分代码，一是在VideoPlayer加载之后加上如下代码：
```bash
    cc.director.setClearColor(cc.color(0, 0, 0, 0)); //清楚颜色的时候清成透明
    let videoList = document.getElementsByClassName("cocosVideo");
    let count = 0;
    for (let idx = 0; idx< videoList.length; idx++){
        let video = videoList[idx];
            video.style.zIndex = count; //设置video的层级
            video.setAttribute("x5-video-player-type", "h5");
            video.setAttribute("webkit-playsinline", true);
            video.setAttribute("x-webkit-airplay", true);
            video.setAttribute("x5-video-orientation", "h5");
            video.setAttribute("x5-video-player-fullscreen", true);
            video.setAttribute("preload", "auto");
            console.log("set video attribute.")
    }
    
    let gCanvas = document.getElementsByClassName('gameCanvas')[0];
    if(gCanvas && false){
        gCanvas.style.position = 'relative';
        this.canvasZIndex = gCanvas.style.zIndex;
        gCanvas.style.zIndex = count + 1;
        console.log("set canvas attribute.")
    }
```
二是在main.js的 function boot (){}里加入如下代码：
```bash
    function boot (){
        .
        .
        .
        var option = {
            //width: width,
            //height: height,
            id: 'GameCanvas',
            scenes: settings.scenes,
            debugMode: settings.debug ? cc.DebugMode.INFO : cc.DebugMode.ERROR,
            showFPS: (!false && !false) && settings.debug,
            frameRate: 60,
            jsList: jsList,
            groupList: settings.groupList,
            collisionMatrix: settings.collisionMatrix,
            renderMode: 0
        }
        //////////////////////////以上是引擎的代码///////////////////////////////////

        cc.macro.ENABLE_TRANSPARENT_CANVAS = true; //加入的代码在这一行，设置允许绘制Canvas为透明

        //////////////////////////以下是引擎的代码///////////////////////////////////
        cc.game.run(option, onStart);
    }
```
因为Canvas盖在了Video上，且Canvas默认是不透明的（为了性能？），不透明的话会遮住底下的元素，所以得设置成可以绘制透明，！当然也会导致一些奇怪的情况，因为设置成了透明。
还需要别的效果就得研究网页的video，设置相关的属性，或者使用别的高级方案。
* 还有比较重要的一点，必须是用户点击屏幕之后才能调用videoPlayer.play()方法播放。否则是不能播放出来的。这个文档上也有说明。而且ios的微信浏览器还要同一帧点击才能播放视频。