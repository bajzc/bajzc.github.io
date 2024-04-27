# 在Windows 11上安装GTK4


![](https://docs.gtk.org/gtk4/hello-world.png){.gtk .gtkalign-center}

# 在Win11上安装GTK4

::: contents
:::

一个关于在Win 11上使用Msys2安装Gtk4,并且使用makefile生成一个Hello
world的demo的指北

并且包括了作者在安装过程中遇到的坑\...

安装过程在的Sandbox里经过测试:

&gt; :::: note
&gt; ::: title
&gt; Note
&gt; :::
&gt;
&gt; Edition Windows 11 Enterprise
&gt;
&gt; Version 22H2
&gt;
&gt; OS build 22621.1848
&gt; ::::
&gt;
&gt; :::: hint
&gt; ::: title
&gt; Hint
&gt; :::
&gt;
&gt; [GTK官网的安装指南](https://www.gtk.org/docs/installations/windows/#using-gtk-from-msys2-packages)
&gt; ::::

## 安装过程

### 1. 下载并安装Msys2

*本教程使用默认安装路径，如使用自定义，在后文设置PATH时需要注意*

从官网下载 [Msys2](https://www.msys2.org/)，并安装

点击Finish后弹出命令行界面，根据Msys2的文档，我们首先安装gcc

&gt; :::: note
&gt; ::: title
&gt; Note
&gt; :::
&gt;
&gt; 国内用户可能需要更换国内源提升下载速度，
&gt;
&gt; 这里我们使用
&gt; [清华大学镜像站](https://mirrors.tuna.tsinghua.edu.cn/help/msys2/)
&gt; ,在终端输入
&gt;
&gt; `sed -i &#34;s#https\?://mirror.msys2.org/#https://mirrors.tuna.tsinghua.edu.cn/msys2/#g&#34; /etc/pacman.d/mirrorlist*`
&gt;
&gt; 完成换源
&gt; ::::

`pacman -S mingw-w64-ucrt-x86_64-gcc`

![image](/images/Msys2_install.png){.align-center}

回车开始安装

:   :::: hint
    ::: title
    Hint
    :::

    在Msys2 Shell里的复制可以快捷键Shift&#43;Ins
    ::::

### 2. 安装GTK4库和toolchain

要安装GTK4，在终端输入

`pacman -S mingw-w64-x86_64-gtk4`

为了在windows上使用GNU/make，输入

`pacman -S mingw-w64-x86_64-toolchain base-devel`

都是一路回车选择默认

&gt; ![image](/images/Msys2_tool-chain.png){.align-center}

### 3. 设置Path

打开
[Path设置](https://www.baidu.com/baidu?ie=utf-8&amp;wd=path%E8%AE%BE%E7%BD%AE)
细节不赘述

在系统变量里找到Path,打开后加上你的安装路径 *mingw64bin*
，（对我来说就是 `C:\msys64\mingw64\bin` ）

再新建一个 `C:\msys64\usr\bin`，点击确定保存。

&gt; :::: note
&gt; ::: title
&gt; Note
&gt; :::
&gt;
&gt; 这里千万要注意顺序，因为Windows会依照PATH从前到后扫描。这对之后的
&gt; `pkg-config` 以及 `mkdir` 都有影响
&gt; ::::

由于我们要使用 `make`
命令,而Msys2默认把它命名为\&#34;mingw32-make\&#34;。所以我们进入刚刚输入的目录（C:/msys64/mingw64/bin），找到
*mingw32-make* 重命名（或者ctrl&#43;c,ctrl&#43;v生成一个副本），命名为make。

此时我们再打开cmd(Win&#43;R)，输入 `make -v` ,发现已经可以运行了。

接着再在cmd中试一下命令 `pkg-config --cflags --libs gtk4`
，如果有类似下面的输出，就说明成功了：

&gt; ![image](/images/cmd_pkg-config.png){.align-center}
&gt;
&gt; :::: hint
&gt; ::: title
&gt; Hint
&gt; :::
&gt;
&gt; 如果没有，那么可以说你不幸遇到了第一个坑,和作者一样（
&gt;
&gt; 首先给MS扣
&gt; [666](https://img.devrant.com/devrant/rant/r_1093122_83dS9.jpg) （菜
&gt;
&gt; 打开Msys2 Shell，运行
&gt;
&gt; `cp -r /mingw64/lib/pkgconfig/* /usr/share/pkgconfig/`
&gt;
&gt; 这就会把它提示的\&#34;gtk4.pc\&#34;以及其他一系列依赖，复制到 `pkg-config`
&gt; 会自动扫描的文件夹下。
&gt; ::::

### 4. Hello World

新建一个文件夹，这里就叫demo,在里面创建demo.c
(Shift&#43;右键文件资源管理器，在该目录下打开powershell，输入
`notepad demo.c` )

这里使用GTK官方的实例：

&gt; ``` C
&gt; #include &lt;gtk/gtk.h&gt;
&gt;
&gt; static void activate(GtkApplication* app, gpointer user_data) {
&gt;     GtkWidget* window;
&gt;
&gt;     window = gtk_application_window_new(app);
&gt;     gtk_window_set_title(GTK_WINDOW(window), &#34;Window&#34;);
&gt;     gtk_window_set_default_size(GTK_WINDOW(window), 200, 200);
&gt;     gtk_widget_show(window);
&gt; }
&gt;
&gt; int main(int argc, char** argv) {
&gt;     GtkApplication* app;
&gt;     int status;
&gt;
&gt;     app = gtk_application_new(&#34;org.gtk.example&#34;, G_APPLICATION_DEFAULT_FLAGS);
&gt;     g_signal_connect(app, &#34;activate&#34;, G_CALLBACK(activate), NULL);
&gt;     status = g_application_run(G_APPLICATION(app), argc, argv);
&gt;     g_object_unref(app);
&gt;
&gt;     return status;
&gt; }
&gt; ```

保存后回到命令行，再创建一个名为 *makefile* 的文件，输入以下内容：

:   ``` makefile
    CC = gcc
    CFLAGS = -O2 -Wall
    GTK_CFLAGS = `pkg-config --cflags gtk4`
    GTK_LIBS = `pkg-config --libs gtk4`

    all:
        @mkdir -p ./out
        $(CC) $(GTK_CFLAGS) -o ./out/demo.exe demo.c $(GTK_LIBS)
    ```

    :::: note
    ::: title
    Note
    :::

    这里的 `mkdir` 又是一个坑，安装Msys2后，在 *usr/bin*
    下会有一个mkdir.exe，它是从linux移植来的，支持linux的一些参数。而powersehll的mkdir则
    [不支持参数](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/mkdir#syntax)
    。所以我们将 *C:msys64usrbin* 加入Path。

    但这可能又会造成一些问题，比如在 *usr/bin* 目录下可能会有
    *python.exe* ，这就可能会影响你使用 `pip`
    安装。所以，PATH的顺序很重要，你可以将python的 */bin*
    目录放在最上面。
    ::::

保存后运行 `make` 命令，exe文件就会再 *./out* 文件夹下生成。

&gt; ![image](/images/GTK_hello_world.png){.align-center}

## 总结

到这里，安装已经基本结束了。可以发现在Windows下安装配置C语言的环境相比Linux来说，体验上差很多。

所以在这里推荐大家使用Gentoo，并使用 `emerge -a gui-libs/gtk` 终结本文（

如果还有其他问题，欢迎在此讨论，有能力者自行Google(Baidu)。

至于使用 *Vcpkg* 安装以及对 *Vistual Studio* 的支持，以后会再补充。


---

> 作者:   
> URL: https://bajzc.org/zh-cn/posts/install-gtk4-on-windows/  

