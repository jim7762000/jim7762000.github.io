---
layout: post
title: Sublime Text 3 in Linux
---

以下文章參考下列四個網址：

1. [完美解决 Linux 下 Sublime Text 中文输入](https://www.sinosky.org/linux-sublime-text-fcitx.html)
2. [How do I make Sublime Text 3 the default text editor](http://askubuntu.com/questions/396938/how-do-i-make-sublime-text-3-the-default-text-editor)

linux上，中文輸入法一直是難題，應用程式沒辦法支援中文輸入是非常常見的事情，連sublime text也是。還不只如此，還有輸入法的戰爭，我在進入linux時，最一開使用的輸入法是gcin，然後去用hime，最後因為sublime text的解決方法只能只用fcitx，最後用了這個輸入法。

我看了不少文章提供各種sublime text輸入的問題，我覺得下列方法是最方便的：

1. 把下列的程式碼存為sublime_imfix.c:


``` c
/*
sublime-imfix.c
Use LD_PRELOAD to interpose some function to fix sublime input method support for linux.
By Cjacker Huang

gcc -shared -o libsublime-imfix.so sublime-imfix.c `pkg-config --libs --cflags gtk+-2.0` -fPIC
LD_PRELOAD=./libsublime-imfix.so subl
*/
#include <gtk/gtk.h>
#include <gdk/gdkx.h>
typedef GdkSegment GdkRegionBox;

struct _GdkRegion
{
  long size;
  long numRects;
  GdkRegionBox *rects;
  GdkRegionBox extents;
};

GtkIMContext *local_context;

void
gdk_region_get_clipbox (const GdkRegion *region,
            GdkRectangle    *rectangle)
{
  g_return_if_fail (region != NULL);
  g_return_if_fail (rectangle != NULL);

  rectangle->x = region->extents.x1;
  rectangle->y = region->extents.y1;
  rectangle->width = region->extents.x2 - region->extents.x1;
  rectangle->height = region->extents.y2 - region->extents.y1;
  GdkRectangle rect;
  rect.x = rectangle->x;
  rect.y = rectangle->y;
  rect.width = 0;
  rect.height = rectangle->height;
  //The caret width is 2;
  //Maybe sometimes we will make a mistake, but for most of the time, it should be the caret.
  if(rectangle->width == 2 && GTK_IS_IM_CONTEXT(local_context)) {
        gtk_im_context_set_cursor_location(local_context, rectangle);
  }
}

//this is needed, for example, if you input something in file dialog and return back the edit area
//context will lost, so here we set it again.

static GdkFilterReturn event_filter (GdkXEvent *xevent, GdkEvent *event, gpointer im_context)
{
    XEvent *xev = (XEvent *)xevent;
    if(xev->type == KeyRelease && GTK_IS_IM_CONTEXT(im_context)) {
       GdkWindow * win = g_object_get_data(G_OBJECT(im_context),"window");
       if(GDK_IS_WINDOW(win))
         gtk_im_context_set_client_window(im_context, win);
    }
    return GDK_FILTER_CONTINUE;
}

void gtk_im_context_set_client_window (GtkIMContext *context,
          GdkWindow    *window)
{
  GtkIMContextClass *klass;
  g_return_if_fail (GTK_IS_IM_CONTEXT (context));
  klass = GTK_IM_CONTEXT_GET_CLASS (context);
  if (klass->set_client_window)
    klass->set_client_window (context, window);

  if(!GDK_IS_WINDOW (window))
    return;
  g_object_set_data(G_OBJECT(context),"window",window);
  int width = gdk_window_get_width(window);
  int height = gdk_window_get_height(window);
  if(width != 0 && height !=0) {
    gtk_im_context_focus_in(context);
    local_context = context;
  }
  gdk_window_add_filter (window, event_filter, context);
}
```

2. Ctrl+Alt+T打開你的Terminal視窗到你儲存上面檔案的地方，鍵入：

``` bash
sudo apt-get install build-essential libgtk2.0-dev
gcc -shared -o libsublime-imfix.so sublime-imfix.c `pkg-config --libs --cflags gtk+-2.0` -fPIC
sudo mv libsublime-imfix.so /opt/sublime_text/
```

這樣就完成編譯，並且將檔案放置到安裝目錄了。

3. 修改啟動部份

``` bash
sudo subl /usr/share/applications/sublime-text.desktop
```

在每一個`Exec=`後面都加上下面的指令：

``` bash
env LD_PRELOAD=/opt/sublime_text/libsublime-imfix.so
```

然後輸入

``` bash
sudo subl /usr/bin/subl
```

更動內容為

``` bash
#!/bin/sh
export LD_PRELOAD=/opt/sublime_text/libsublime-imfix.so
exec /opt/sublime_text/sublime_text "$@"
```

4. 如果想要把sublime text更動為預設編輯器，先使用下列指令確定是否有安裝成功：

``` bash
ls /usr/share/applications/sublime-text.desktop
```

接著打開linux的default列表：

``` bash
sudo subl /usr/share/applications/defaults.list
```

按下Ctrl+H replace gedit with sublime-text。接著打開user的設定列表：

``` bash
subl ~/.local/share/applications/mimeapps.list
```

修改或添加下列下列文字：

``` bash
[Added Associations]
text/plain=ubuntu-software-center.desktop;shotwell.desktop;sublime-text.desktop;
text/x-chdr=shotwell-viewer.desktop;

[Default Applications]
text/plain=sublime-text.desktop
text/x-c++src=sublime-text.desktop
text/x-chdr=sublime-text.desktop
```

