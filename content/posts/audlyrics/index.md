+++
title = "Audlyrics"
date = "2023-02-06T18:21:30+08:00"
author = ""
authorTwitter = "" #do not include @
cover = ""
tags = ["music"]
keywords = ["audacious", "rust", "KDE"]
description = "Audacious 的歌词工具 on KDE Plasmashell"
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
+++

## 动机

偶然发现一直在用的音乐播放器 Audacious 附带了一个命令行工具 `audtool` ，可以控制播放和获取播放信息。

这东西好啊，一直以来在 Plasmashell 上显示歌词的执念终于可以满足了。

## 技术路线

说起 Plasmashell 当然得是 Plamoid ，写 Plasmoid 当然得用 QML 。
但是吧， QML 框架好像不能直接访问文件系统，也不能执行命令，而要做到这些就得用 C++ ，这就很头疼了。

所以思索之下，我最终选择了用 Rust 写一个服务器，然后 QML 只负责展示。

Rust 的服务器框架随便选了个 Hyper 。

## 实现

思路很简单，在服务端开子进程执行 `audtool` ，获取当前曲目位置和播放时间，刚好我的歌词就存在曲目边上，和音频文件同名。

从歌词文件里读出来 LRC 格式的文本，然后解析后存起来，前端轮询的时候根据播放时间返回当前句子就行了。

QML 懒得学，但是刚好看见过一个叫 `ypm-lyrics` 的 Plasmoid ，这个是用来显示网易云第三方客户端 YesplayMusic 的歌词的，就直接拿来用了。

在和 Rust 的异步生命周期检查搏斗许久之后，终于搞定了服务端。

从中总结出来的经验是：闭包写短点，要不然容易晕。。。（汗

然后为了无感启动和关闭服务端，又为了不想让服务端一直开着（虽然基本不会占资源），在启动服务端时会把 Audacious 作为子进程启动，然后守护 Audacious 退出。

{{< code language="rust" title="服务器核心代码" id="1" expand="Show" collapse="Hide" >}}

    let mut child = Command::new("audacious").spawn()
        .expect("Cannot start audacious. Is it Installed?");


    let state = Arc::new(Mutex::new(State::new()));

    let make_service = make_service_fn(move |_: &AddrStream| {
        let state = state.clone();

        let service = service_fn(
        move |req: Request<Body>| {
            handle(state.clone(), req)
        });

        async move {
            Ok::<_, Infallible>(service)
        }
    });

    let addr = ([127, 0, 0, 1], 30123).into();
    let server = Server::bind(&addr).serve(make_service);

    let grace = server.with_graceful_shutdown(async move {
        match child.wait().await {
            Ok(code) => println!("Audacious returned with {}", code),
            Err(e) => println!("Audacious returned with IoError {:?}", e)
        };
    });

    if let Err(e) = grace.await {
        eprintln!("Server error: {}", e);
    }
{{< /code >}}

端口直接写死了。

{{< emoji "bailan.jpg" >}}

## 效果

{{< figure src="playing.png" alt="Playing" position="center" caption="Playing" captionPosition="right" >}}

{{< figure src="paused.png" alt="Paused" position="center" caption="Paused" captionPosition="right" >}}


源代码可以在 [Github](https://github.com/danzou1ge6/audlyrics/) 找到。
