# 主要示例介绍



终于到了本书中编写代码的部分。

Node的事件循环是经过多年发展的一个复杂软件。我们会简化很多部分。

我们将通过实现Node中比较重要的部分，从而更好地理解Node。更重要的是，我们能够运用在之前章节中学到的知识来构建一个真正能够运行的程序。

我们的目标是探索异步的概念。而拿Node举例纯粹是为了趣味性。

**理想中的代码成果：**

```rust, no_run
/// Think of this function as the javascript program you have written
/// 把这个函数想象成为 就是你写过的JavaScript程序
fn javascript() {
    print("First call to read test.txt");
    Fs::read("test.txt", |result| {
        let text = result.into_string().unwrap();
        let len = text.len();
        print(format!("First count: {} characters.", len));

        print(r#"I want to create a "magic" number based on the text."#);
        Crypto::encrypt(text.len(), |result| {
            let n = result.into_int().unwrap();
            print(format!(r#""Encrypted" number is: {}"#, n));
        })
    });

    print("Registering immediate timeout 1");
    set_timeout(0, |_res| {
        print("Immediate1 timed out");
    });
    print("Registering immediate timeout 2");
    set_timeout(0, |_res| {
        print("Immediate2 timed out");
    });

    // let's read the file again and display the text
    // 再次读取文件，展示内容。
    print("Second call to read test.txt");
    Fs::read("test.txt", |result| {
        let text = result.into_string().unwrap();
        let len = text.len();
        print(format!("Second count: {} characters.", len));

        // aaand one more time but not in parallel.
        print("Third call to read test.txt");
        Fs::read("test.txt", |result| {
            let text = result.into_string().unwrap();
            print_content(&text, "file read");
        });
    });

    print("Registering a 3000 and a 500 ms timeout");
    set_timeout(3000, |_res| {
        print("3000ms timer timed out");
        set_timeout(500, |_res| {
            print("500ms timer(nested) timed out");
        });
    });

    print("Registering a 1000 ms timeout");
    set_timeout(1000, |_res| {
        print("SETTIMEOUT");
    });

    // `http_get_slow` lets us define a latency we want to simulate
    // `http_get_slow` 提供了接口，供我们设定模拟的延迟。
    print("Registering http get request to google.com");
    Http::http_get_slow("http//www.google.com", 2000, |result| {
        let result = result.into_string().unwrap();
        print_content(result.trim(), "web call");
    });
}

fn main() {
    let rt = Runtime::new();
    rt.run(javascript);
}
```

> 之后我们会在代码中的关键点设置输出语句，这样我们就能看到何时何地发生了什么。

你马上就会看到我们在示例中编写的Rust代码有些奇怪。

当我们希望执行如同JavaScript那样的异步操作时，会使用到回调函数；正如你在Node中导入模块一样，我们也可能调用像`Fs`和`Crypto`这样的“魔法”模块。

这部分的代码主要是调用函数——注册事件，以及存储一个在事件就绪时需要被执行的回调函数。

**`set_time`函数的一个用例：**

```rust, no_run
set_timeout(0, |_res| {
    print("Immediate1 timed out");
});
```

这一步的意思就是：我们注册了对`timeout`事件的兴趣，接着当事件发生时，我们就要运行回调函数``|_res| { print("Immediate1 timed out"); }`。

这里`_res`是传递给回调函数的参数。在这个例子中，JavaScript会省略掉这个参数，如`setTimeout(()=>{console.log("Immediate1 timed out");}, 0)`；

而既然我们现在使用的Rust是一门强类型的语言，那么不妨创建一种名为`Js`的类型。

`Js`是一种枚举类型，用于表示JavaScript类型。在给`set_timeout`传递回调时，`_res`的具体类型是`Js::undefined`。给`Fs::read`传递回调时，`_res`的具体类型就是`Js::String`等。

运行这段代码，会得到以下输出：

<video autoplay loop>
<source src="./images/example_run.mp4" type="video/mp4">
Can't display video.
</video>
程序的输出：

```
Thread: main     First call to read test.txt
Thread: main     Registering immediate timeout 1
Thread: main     Registered timer event id: 2
Thread: main     Registering immediate timeout 2
Thread: main     Registered timer event id: 3
Thread: main     Second call to read test.txt
Thread: main     Registering a 3000 and a 500 ms timeout
Thread: main     Registered timer event id: 5
Thread: main     Registering a 1000 ms timeout
Thread: main     Registered timer event id: 6
Thread: main     Registering http get request to google.com
Thread: pool3    received a task of type: File read
Thread: pool2    received a task of type: File read
Thread: main     Event with id: 7 registered.
Thread: main     ===== TICK 1 =====
Thread: main     Immediate1 timed out
Thread: main     Immediate2 timed out
Thread: pool3    finished running a task of type: File read.
Thread: pool2    finished running a task of type: File read.
Thread: main     First count: 39 characters.
Thread: main     I want to create a "magic" number based on the text.
Thread: pool3    received a task of type: Encrypt
Thread: main     ===== TICK 2 =====
Thread: main     SETTIMEOUT
Thread: main     Second count: 39 characters.
Thread: main     Third call to read test.txt
Thread: main     ===== TICK 3 =====
Thread: pool2    received a task of type: File read
Thread: pool3    finished running a task of type: Encrypt.
Thread: main     "Encrypted" number is: 63245986
Thread: main     ===== TICK 4 =====
Thread: pool2    finished running a task of type: File read.

===== THREAD main START CONTENT - FILE READ =====
Hello world! This is text to encrypt!
... [Note: Abbreviated for display] ...
===== END CONTENT =====

Thread: main     ===== TICK 5 =====
Thread: epoll    epoll event 7 is ready

===== THREAD main START CONTENT - WEB CALL =====
HTTP/1.1 302 Found
Server: Cowboy
Location: http://http/www.google.com
... [Note: Abbreviated for display] ...
===== END CONTENT =====

Thread: main     ===== TICK 6 =====
Thread: epoll    epoll event timeout is ready
Thread: main     ===== TICK 7 =====
Thread: main     3000ms timer timed out
Thread: main     Registered timer event id: 10
Thread: epoll    epoll event timeout is ready
Thread: main     ===== TICK 8 =====
Thread: main     500ms timer(nested) timed out
Thread: pool0    received a task of type: Close
Thread: pool1    received a task of type: Close
Thread: pool2    received a task of type: Close
Thread: pool3    received a task of type: Close
Thread: epoll    received event of type: Close
Thread: main     FINISHED
```

别担心，我们会慢慢解释这一切的，这里我只是想先介绍一下最终能够达到的效果。

