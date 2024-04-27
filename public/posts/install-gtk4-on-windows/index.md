# Install GTK4 on Windows


![](https://docs.gtk.org/gtk4/hello-world.png){.gtk .gtkalign-center}

# Install GTK4 on Windows11

::: contents
:::

This is a guide to install and config GTK4 on Windows 11.

&gt; :::: note
&gt; ::: title
&gt; Note
&gt; :::
&gt;
&gt; All steps have been tested in Sandbox:
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
&gt; [GTK official
&gt; guide](https://www.gtk.org/docs/installations/windows/#using-gtk-from-msys2-packages)
&gt; ::::

## 1. Download and install Msys2

*This tutorial uses the default installation path, if you use a custom
one, you need to pay attention when setting PATH later*

Download from [here](https://www.msys2.org/), and install

After clicking Finish, the command line interface will pop up. According
to the documentation of Msys2, we first install gcc:

`pacman -S mingw-w64-ucrt-x86_64-gcc`

Enter(Y) to install

:   ![image](/images/Msys2_install.png){.align-center}

    :::: hint
    ::: title
    Hint
    :::

    Shift&#43;Ins is the shortcut to paste in Msys2 Shell.
    ::::

## 2. Install GTK4 and Mingw-toolchain

In order to install GTK4, type in the terminal:

`pacman -S mingw-w64-x86_64-gtk4`

To use GNU/make and other tools, type:

`pacman -S mingw-w64-x86_64-toolchain base-devel`

Enter until finished.

:   ![image](/images/Msys2_tool-chain.png){.align-center}

## 3. Setting Path

Edit System Variable (Follow
[here](https://www.java.com/en/download/help/path.html))

Add *Your path to msys2mingw64bin*, as default it\&#39;s
`C:\msys64\mingw64\bin`.

Add *Your path to msys2usrbin*, as default it\&#39;s `C:\msys64\usr\bin`.

:::: note
::: title
Note
:::

Follow the sequence above, as Windows will follow the sequence on PATH
table.

This will affect our latter input commands `pkg-config` and `mkdir`.
::::

Given that we need to use `make`, however, Msys2 will name it as
\&#34;mingw32-make.exe\&#34;.

So we go to the directory we just put in PATH (`C:\msys64\mingw64\bin`),
find the \&#34;mingw32-make.exe\&#34; and rename to \&#34;make.exe\&#34; (or create
copy).

Now, when we open `cmd` or `powershell`, and type `make -v`, it works!

&gt; ![image](/images/cmd_make.png){.align-center}
&gt;
&gt; Then, we try `pkg-config --cflags --libs gtk4`. We expect to have
&gt; output like this:
&gt;
&gt; ![image](/images/cmd_pkg-config.png){.align-center}
&gt;
&gt; :::: hint
&gt; ::: title
&gt; Hint
&gt; :::
&gt;
&gt; If not\... [\&#34;Follow\&#34;
&gt; this](https://img.devrant.com/devrant/rant/r_1093122_83dS9.jpg)
&gt;
&gt; then, open Msys2 shell, run
&gt;
&gt; `cp -r /mingw64/lib/pkgconfig/* /usr/share/pkgconfig/`
&gt;
&gt; to copy \&#34;gtk4.pc\&#34; to the path that `pkg-config` can find.
&gt; ::::

## 4. Hello world

Open a new folder, and create a new file called `demo.c`.

We use the demo from [GTK](gtk.org):

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

And another file called `makefile`:

&gt; ``` makefile
&gt; CC = gcc
&gt; CFLAGS = -O2 -Wall
&gt; GTK_CFLAGS = `pkg-config --cflags gtk4`
&gt; GTK_LIBS = `pkg-config --libs gtk4`
&gt;
&gt; all:
&gt;     @mkdir -p ./out
&gt;     $(CC) $(GTK_CFLAGS) -o ./out/demo.exe demo.c $(GTK_LIBS)
&gt; ```

Remember to save them, and open a `cmd` in this directory (Shift &#43; Right
click).

Run `make`, the .exe file will be generated under *./out*. (Ignore
warning message)

&gt; ![image](/images/GTK_hello_world.png){.align-center}

# Summary

That\&#39;s all the steps, looks like it\&#39;s hard to config the basic
environment for C development.

So, life is short, I choose Linux :)


---

> Author:   
> URL: https://bajzc.org/posts/install-gtk4-on-windows/  

