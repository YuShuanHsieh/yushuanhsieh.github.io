---
layout: single
title:  "Discussing the Action Manager in Cocos2d-x"
date:   2017-09-15 14:18:11 +0100
categories: Cocos2d-x
---
**[前言]**  
前陣子才剛從英國完成論文返回台灣。因為忙論文 / 忙搬家 / 忙調作息，所以好些時間沒有更新相關知識，覺得自己好像停滯了...明天就是轉職人生的開始，趕緊來繼續惡補Cocos2d-x知識。API使用方式和理解固然很重要，不過我也相當喜歡探討一些運作流程和結構方式，所以文章內容也會比較著重在這部分。這次來看的是有關於node->runAction(Action* action);之前我寫遊戲時使用完全不同的方式來讓遊戲像素變動，因此會比較想了解引擎是如何讓node根據我們設定的動作來變化。

**[涉及Class]**  
`ActionManager`  
`UT_hash_handle`  

**[Cocos2d-x Main Loop]**  
在討論ActionManager之前，先來看一下Cocos2d-x的遊戲主循環：  
![Layout Screenshot]({{ site.url }}/assets/images/mainloop.png)  
從圖中可以看到main loop中透過Schedule先將特定物件(在這邊為nodes)的圖像數據update，然後再繪製出調整後的nodes，這樣就能夠呈現出物品移動的視覺效果了。  

**[Action Manager]**  
要讓node產生變化ActionManager和Schedule是非常重要的兩個Class，ActionManager負責管理所有nodes的actions，Schedule則是負責所有物件的update排程。當Director初始化的時候，會產生出ActionManager，並將ActionManager預先排入Schedule中，這樣每次在遊戲迴圈中，就會執行ActionManager 的update function來執行其底下所管理的動作。
```
_scheduler = new (std::nothrow) Scheduler();
_actionManager = new (std::nothrow) ActionManager();
_scheduler->scheduleUpdate(_actionManager, Scheduler::PRIORITY_SYSTEM, false);
/* Scheduler::PRIORITY_SYSTEM 高優先權 */
```

至於ActionManager是如何管理各個node的動作呢？當使用者寫node->runAction(Action* action);的時候，其實就是把node和指定動作一起加入到ActionManager中。  
```
Action * Node::runAction(Action* action)
{
    CCASSERT( action != nullptr, "Argument must be non-nil");
    _actionManager->addAction(action, this, !_running);
    return action;
}
```

ActionManager使用HashTable List來管理這些node，其struct為：  
```
typedef struct _hashElement
{
    struct _ccArray     *actions;
    Node                *target;
    int                 actionIndex;
    Action              *currentAction;
    bool                currentActionSalvaged;
    bool                paused;
    UT_hash_handle      hh;
} tHashElement;
```
這邊可以看到 _hashElement 中包含了node(*target)和action arrays(*actions)可以保存node以及對應的action sequence，而UT_hash_handle則是cocos2d自訂的資料結構，其中包含  *previous 和 *next 供element search，另外還有key值。當ActionManager執行update function的時候：  
1. visit _hashElement(first loop)
2. 從actions中取出currentAction並執行。  
```
_currentTarget->currentAction->step(dt);
```
3. remove current action.
4. 如果target的動作全部執行完畢，則remove _hashElement(*target)

這樣看下來就很清楚ActionManager是如何運作的了！
