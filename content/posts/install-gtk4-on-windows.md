---
category: |
  Tutorial
date: "2023-07-11 12:39:46 UTC+07:00"
lang: |
  en
slug: |
  install-gtk4-on-windows
tags: ["GTK4", "Msys2", "Windows", "Installation", "GNU/make"]
title: Install GTK4 on Windows
translation_id: |
  install-gtk4-on-windows
---

![](https://docs.gtk.org/gtk4/hello-world.png){.gtk .gtkalign-center}

# Install GTK4 on Windows11

::: contents
:::

This is a guide to install and config GTK4 on Windows 11.

> :::: note
> ::: title
> Note
> :::
>
> All steps have been tested in Sandbox:
>
> Edition Windows 11 Enterprise
>
> Version 22H2
>
> OS build 22621.1848
> ::::
>
> :::: hint
> ::: title
> Hint
> :::
>
> [GTK official
> guide](https://www.gtk.org/docs/installations/windows/#using-gtk-from-msys2-packages)
> ::::

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

    Shift+Ins is the shortcut to paste in Msys2 Shell.
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

Add *Your path to msys2mingw64bin*, as default it\'s
`C:\msys64\mingw64\bin`.

Add *Your path to msys2usrbin*, as default it\'s `C:\msys64\usr\bin`.

:::: note
::: title
Note
:::

Follow the sequence above, as Windows will follow the sequence on PATH
table.

This will affect our latter input commands `pkg-config` and `mkdir`.
::::

Given that we need to use `make`, however, Msys2 will name it as
\"mingw32-make.exe\".

So we go to the directory we just put in PATH (`C:\msys64\mingw64\bin`),
find the \"mingw32-make.exe\" and rename to \"make.exe\" (or create
copy).

Now, when we open `cmd` or `powershell`, and type `make -v`, it works!

> ![image](/images/cmd_make.png){.align-center}
>
> Then, we try `pkg-config --cflags --libs gtk4`. We expect to have
> output like this:
>
> ![image](/images/cmd_pkg-config.png){.align-center}
>
> :::: hint
> ::: title
> Hint
> :::
>
> If not\... [\"Follow\"
> this](https://img.devrant.com/devrant/rant/r_1093122_83dS9.jpg)
>
> then, open Msys2 shell, run
>
> `cp -r /mingw64/lib/pkgconfig/* /usr/share/pkgconfig/`
>
> to copy \"gtk4.pc\" to the path that `pkg-config` can find.
> ::::

## 4. Hello world

Open a new folder, and create a new file called `demo.c`.

We use the demo from [GTK](gtk.org):

> ``` C
> #include <gtk/gtk.h>
>
> static void activate(GtkApplication* app, gpointer user_data) {
>     GtkWidget* window;
>
>     window = gtk_application_window_new(app);
>     gtk_window_set_title(GTK_WINDOW(window), "Window");
>     gtk_window_set_default_size(GTK_WINDOW(window), 200, 200);
>     gtk_widget_show(window);
> }
>
> int main(int argc, char** argv) {
>     GtkApplication* app;
>     int status;
>
>     app = gtk_application_new("org.gtk.example", G_APPLICATION_DEFAULT_FLAGS);
>     g_signal_connect(app, "activate", G_CALLBACK(activate), NULL);
>     status = g_application_run(G_APPLICATION(app), argc, argv);
>     g_object_unref(app);
>
>     return status;
> }
> ```

And another file called `makefile`:

> ``` makefile
> CC = gcc
> CFLAGS = -O2 -Wall
> GTK_CFLAGS = `pkg-config --cflags gtk4`
> GTK_LIBS = `pkg-config --libs gtk4`
>
> all:
>     @mkdir -p ./out
>     $(CC) $(GTK_CFLAGS) -o ./out/demo.exe demo.c $(GTK_LIBS)
> ```

Remember to save them, and open a `cmd` in this directory (Shift + Right
click).

Run `make`, the .exe file will be generated under *./out*. (Ignore
warning message)

> ![image](/images/GTK_hello_world.png){.align-center}

# Summary

That\'s all the steps, looks like it\'s hard to config the basic
environment for C development.

So, life is short, I choose Linux :)
