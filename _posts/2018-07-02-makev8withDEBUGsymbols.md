---
title: 编译带 debug symbols 的 V8
date: 2018-07-02 18:14
---

> 因为没钱买国外服务器的缘故，源码拖得比较苦，刚开始有点难度，现在比较知道怎么对付。
                                                                                  ——李宗盛

最近有需要编译旧版 v8。v8 的 github 上写说，直接 git clone 下来然后编译是不行的，需要使用 Google 的版本控制工具 depot tools 来 fetch 源码。但由于某些无言的网络原因，fetch 常常会中断，原因出在 depot tools 会去 Google Drive 上取一些存有 sha1 的文件，而这些文件的链接都以 `gs://xx` 为开头，使用代理（譬如 proxychains）下载时会奇怪地中断掉。具体报错像是：
```
________ running 'download_from_google_storage --no_resume --platform=linux* --no_auth --bucket chromium-clang-format -s v8/buildtools/linux64/clang-format.sha1' in '/home/v

8/source'
Failed to fetch file gs://chromium-clang-format/5349d1954e17f6ccafb6e6663b0f13cdb2bb33c8 for v8/buildtools/linux64/clang-format. [Err: /home/v8/depot_tools/vpython: line 42:
 /home/v8/depot_tools/.cipd_bin/vpython: No such file or directory
```

之类的。

下载失败的文件链接会在 fetch 失败时间过长后 print 出来，理论上讲，可以自己手动把这些文件下载下来，然后在 `gclient sync` 一下就好了，但实际不行，有一些小小的限制。
这个下载模块的实现主要是在 depot tools 下的 `download_from_google_storage.py` ，里边有两个可以注意的地方：
第一个地方是 `base_url = 'gs://%s' % options.bucket` 这一行。bucket 就是它要去 Google Drive 上所取的文件，理论上说把这个 gs 改成 https://storage.googleapis.com/ 就行了。然而并不行，可能是我本地代理的问题。
第二个地方是 `downloader_worker_thread` 这个函数，它实现了下载线程，函数有一个参数 delete，默认为 true，作用是：如果需要下载的文件已经存在，就删除掉文件再重新去拿一次，这里需要改成 false。之后自己手动去下载文件， 再`gclient sync` 就好。不过这样相对比较麻烦，而且后来发现有一个问题，即便自己手动地把文件下载下来，depot tools 也是不认的，提示说有改动你需要 git commit 或回滚云云，当然这段大约可以改改 gclient 什么的。
另一个办法是在这里边自己写点代码，去代替它的下载函数，可以简单的用 `urllib2` 去下载，target 直接用 file_url，文件位置 ouput_filename 就行。但是试了下也不行，原因不明……
总而言之后面放弃了使用官方的工具，决定直接把源码拖下来然后按照平常的办法编译。发现能编……步骤如下：
- 1，到 https://chromium.googlesource.com/v8/v8/+refs 拿想要编译的版本的 tar 档。
- 2，`make dependencies`
https://chromium.googlesource.com/v8/v8/+/3c660e485ea372d1076aecdcece69842563d6adf/Makefile
某些 Makefile 里边的依赖项目比较久远，有些地址已经被抛弃的，就需要到 https://chromium.googlesource.com/?format=HTML 上去找对应的东西，像上面这个 3c660e blabla 的版本就需要改成：
```
        git clone https://chromium.googlesource.com/chromium/deps/icu46 \
                third_party/icu
        git clone https://chromium.googlesource.com/external/gyp \
                build/gyp
```
- 3，`make x64.debug library=shared snapshot=on disassembler=on gdbjit=on debuggersupport=on `
- 4, 把 `lib.target`下的所有 so 丢进 `/usr/lib/`。
- 5, 可以用 `gdb --args ./d8`  然后 `r xx.js`开始调试了，tools 底下有 gdbinit 和 gdb-v8-support.py 可用，某些版本没有 gdbinit。可以去找一份有的，然后复制下来进行调试，没啥大区别。

<del>要是有钱该多好，在服务器上编译完再直接拉回本地就行了……如果有富婆/土豪看到这句话的话最好可以联系我资助我谢谢。</del>
