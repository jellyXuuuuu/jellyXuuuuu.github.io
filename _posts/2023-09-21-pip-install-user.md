---
layout: post
title: Git Install Error
---

Q: Could not install packages due to an OSError: [WinError 2] No such file or directory

```shell
ERROR: Could not install packages due to an OSError: [WinError 2] The system cannot find the file specified: 'c:\\python39\\Scripts\\geocode.exe' -> 'c:\\python39\\Scripts\\geocode.exe.deleteme'
```

A:
```shell
pip install numpy --user
```

链接参考：
[https://stackoverflow.com/questions/66322049/could-not-install-packages-due-to-an-oserror-winerror-2-no-such-file-or-direc](https://stackoverflow.com/questions/66322049/could-not-install-packages-due-to-an-oserror-winerror-2-no-such-file-or-direc)


