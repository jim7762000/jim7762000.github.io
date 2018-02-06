---
layout: post
title: How to use xelatex in sublime text
---

To setup LatexTools in sublime text.

You should install package control before installation of LaTexTools. Please go to the official website of package control and copy the commend of installation. Then open console in sublime text via `` ctrl+` `` and paste the commend copied from website. Now, you can start to install LatexTools in sublime text. Call install commend via `ctrl+shift+p` and key in `install package`. Enter `LaTexTools` to install.

To use LaTexTools, you should have MikTex and SumatraPDF in windows OS or texlive and latexmk in linux. If you use windows, you should guarantee that path consist the path to `xetex.exe` and `SumatraPDF.exe`. Next, configure the settings of LaTexTools found in `Preferences -> Package Settings -> LaTexTools` and migrate settings (Select `Reconfigure LaTexTools and migrate settings`) first. Finally, select `Settings-User` and set the builder_settings as following.

``` bash
"builder_settings" : {
  "program": "xelatex",
  // General settings:
  // See README or third-party documentation

  // (built-ins): true shows the log of each command in the output panel
  "display_log" : false,
  "command": ["texify", "-b", "-p", "--engine=xetex", "--tex-option=\"--synctex=1\""],

  // Platform-specific settings:
  "osx" : {
    // See README or third-party documentation
  },

  "windows" : {
    // See README or third-party documentation
  },

  "linux" : {
    // See README or third-party documentation
  }
}
```


