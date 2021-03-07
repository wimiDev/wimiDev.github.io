## 注意事项：

参考的版本：cocos2d-x 3.17.2

## 什么才是DrawCall

Draw Call也就是GPU执行一次绘制指令，它可以是绘制一个三角形或者很多三角形。

## 怎么样才能减少DrawCall

1. 尽量利用自动渲染合批：

自动合批需要满足以下条件。

a、是在同一个渲染分支的通一组命令（globalZ一致）。

b、在同一组globalZ里，且 command::type==TRIANGLES_COMMAND且连续，不然合批就会被打断。

c、MaterialID要相同，如果不同在Renderer::drawBatchedTriangles()还是会被拆散。

\2. 使用显式合批（SpriteBatchNode）

用SpriteBatchNode来实现那些大批量的节点显示，我觉得如果有大批量可预测的精灵要绘制的时候可以显示的使用这个方案。因为它更精准一些，不会被别的渲染命令插队，绘制的时候肯定是合批的。

3.把在屏幕外或者不需要显示的节点的visible设置成false

visible=false就不会进渲染队列，在迭代的时候就会被丢弃掉，并不渲染。但是哪怕这个节点的visiable=false也还是会进渲染树，遍历的时候还是会访问到，对GPU来说确实没消耗，但是对CPU有消耗。更好的方案是，大批量不需要渲染的物体或者不在可是范围内的物体可以先在内存存起来，不进渲染树就不会产生遍历渲染树的消耗。

## 为什么这样就能减少DrawCall

这里会涉及到详细的渲染流程，下面先从mianloop开始依次分析。

### MainLoop

下面是mainloop主要体现的是整个引擎的执行流程。流程大概是判断是否退出循环，是否重启，如果都没有就开始处理和绘制当前场景，执行内存回收。

```cpp
void Director::mainLoop()
{
    if (_purgeDirectorInNextLoop)
    {
        //退出的时候关掉整个GLView
        _purgeDirectorInNextLoop = false;
        purgeDirector();
    }
    else if (_restartDirectorInNextLoop)
    {
        //下一帧重启，当重启标志位=true
        _restartDirectorInNextLoop = false;
        restartDirector();
    }
    else if (!_invalid)
    {
        //开始绘制场景
        drawScene();
     
        // release the objects 
        // 执行一次内存回收
        PoolManager::getInstance()->getCurrentPool()->clear();
    }
}
```

### DrawScene

DrawScene这个方法负责，处理玩家输入，刷新游戏世界，绘制游戏世界。处理的流程大致如图：

![img](images\drawcall\渲染总流程.jpg)

```text
// Draw the Scene
void Director::drawScene()
{
    // calculate "global" dt
    calculateDeltaTime();
    
    if (_openGLView)
    {
        _openGLView->pollEvents();
    }

    //tick before glClear: issue #533
    if (! _paused)
    {
        _eventDispatcher->dispatchEvent(_eventBeforeUpdate);
        _scheduler->update(_deltaTime);
        _eventDispatcher->dispatchEvent(_eventAfterUpdate);
    }

    _renderer->clear();
    experimental::FrameBuffer::clearAllFBOs();
    
    _eventDispatcher->dispatchEvent(_eventBeforeDraw);
    
    /* to avoid flickr, nextScene MUST be here: after tick and before draw.
     * FIXME: Which bug is this one. It seems that it can't be reproduced with v0.9
     */
    if (_nextScene)
    {
        setNextScene();
    }

    pushMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_MODELVIEW);
    
    if (_runningScene)
    {
#if (CC_USE_PHYSICS || (CC_USE_3D_PHYSICS && CC_ENABLE_BULLET_INTEGRATION) || CC_USE_NAVMESH)
        _runningScene->stepPhysicsAndNavigation(_deltaTime);
#endif
        //clear draw stats
        _renderer->clearDrawStats();
        
        //render the scene
        if(_openGLView)
            _openGLView->renderScene(_runningScene, _renderer);
        
        _eventDispatcher->dispatchEvent(_eventAfterVisit);
    }

    // draw the notifications node
    if (_notificationNode)
    {
        _notificationNode->visit(_renderer, Mat4::IDENTITY, 0);
    }

    updateFrameRate();
    
    if (_displayStats)
    {
#if !CC_STRIP_FPS
        showStats();
#endif
    }
    //为什么这里还要执行一遍render
    _renderer->render();

    _eventDispatcher->dispatchEvent(_eventAfterDraw);

    popMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_MODELVIEW);

    _totalFrames++;

    // swap buffers
    if (_openGLView)
    {
        _openGLView->swapBuffers();
    }

    if (_displayStats)
    {
#if !CC_STRIP_FPS
        calculateMPF();
#endif
    }
}
```

这里的主要流程是，刷新游戏世界，然后遍历场景，再渲染场景。

### Render 核心渲染器

Render核心的功能是渲染，它实现的功能包括管理渲染命令，对渲染命令进行整理排序再渲染处理好的命令。达到正确渲染场景的效果。下面是Render的基本结构

![img](images\drawcall\cocos2d-x 3.x Render.jpg)

Render主要维护*renderGroups<RenderQueue>(渲染分支),* commomandGroupStack<int>（当前正在操作哪个分支）, queuedTriangleCommomands<TrianglesCommand*>（批量绘制三角形）这几个缓冲区，通过这些缓冲区来实现渲染命令的排序和渲染。

RenderQueue也称作渲染分支，通常情况下*renderGroups里只有一个分支，在一些特殊的渲染需求下会有多个，比如渲染ClippedNode的时候就会有多个渲染分支。*

![img](images\drawcall\v2-e3b62204d358d4ca02fe5d62088fb753_720w.png)

### 1、排序

在处理渲染命令之前会先对渲染分支进行排序，排序的依据是：

```text
static bool compareRenderCommand(RenderCommand* a, RenderCommand* b)
{
    //对于2d的渲染
    return a->getGlobalOrder() < b->getGlobalOrder();
}

static bool compare3DCommand(RenderCommand* a, RenderCommand* b)
{
    //对于3D的渲染
    return  a->getDepth() > b->getDepth();
}
```

### 2、渲染主干分支

一个渲染分支按照渲染的实现来分，又可以分为5种渲染命令，各自成一组：

![img](images\drawcall\v2-7a2d26122dacc63afecfcdbe6b87d346_720w.jpg)

每一组的含义如下：

GLOBALZ_NEG: globalZ< 0的命令

OPAQUE_3D：globalZ=0的不透明3D命令

TRANSPARENT_3D：globalZ=0的3D透明命令

GLOBALZ_ZERO：globalZ=0的2D命令

GLOBALZ_POS：globalZ>0的命令

接下来按照枚举值0--4依次处理这些命令组（5个commands[]）,按照顺序的处理是出自于图形学的原因，因为有透明的物体存在，就不再详细这个过程了。

### 3、处理渲染命令队列（RenderCommands）

RenderCommand命令也分成多种不同的类型，每一种的处理都有所不同，所有类型的枚举如下：

![img](images\drawcall\v2-da2b78a758226ed17bb7676b526ff716_720w.jpg)



- QUAD_COMMAND：四边形渲染命令，当前未使用
- CUSTOM_COMMAND：CustomCommand，自定义渲染命令
- BATCH_COMMAND：BatchCommand，同时渲染多个使用同一纹理的图形，提高渲染效率
- GROUP_COMMAND：GroupCommand，创建渲染分支，使用_renderGroups[0]之外的RenderQueue
- MESH_COMMAND：MeshCommand，渲染网格
- PRIMITIVE_COMMAND：PrimitiveCommand，渲染自定义图元。
- **TRIANGLES_COMMAND**：TrianglesCommand，渲染三角形，这个命令不会直接执行渲染，而是加到一个三角形渲染队列里面，方便执行批量渲染。可以合并命令减少OpenGL的调用提高渲染效率。但是这里有几个条件：1、是在同一个渲染分支的通一组命令（globalZ一致）。2、在同一组globalZ里 command::type==TRIANGLES_COMMAND且连续，不然就会被打断。3、currentMaterialID要相同，如果不同在Renderer::drawBatchedTriangles()还是会被拆散。