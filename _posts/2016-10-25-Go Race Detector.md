---
layout: post
title: GO语言竞态条件检测
date: 2016-10-25
comments: true
archive: true
tag: [GO, race]
---
多线程编程是当下程序开发的大势所趋，GO语言也不例外，尤其GO语言以擅长并发编程为特色。多线程并发编程中容易出现的一类问题就是竞态条件，它可能产生严重且匪夷所思的问题。幸运的是GO提供了一个简单且有效的竞态条件检测工具，能够在一定程度上检测出来程序中存在静态条件问题。

### 竞态条件检测工具的使用
竞态条件检测工具与GO工具链集成在一起，只需要在命令行中加上-race参数即可。例如：

```
$ go test -race mypkg    // test the package
$ go run -race mysrc.go  // compile and run the program
$ go build -race mycmd   // build the command
$ go install -race mypkg // install the package
```

当在命令行中加上-race参数时，在程序运行的过程中，GO相关工具会记录不同线程对共享变量的访问情况，如果发现非同步的访问则会退出并打印告警信息。需要注意的是加上-race参数之后，相比于不加-race参数，CPU和内存的消耗约上升10倍，所以在真实的环境中并不适合使用竞态条件检测。另外，竞态条件检测并不能完全检测出程序中可能存在的竞态条件，只有在程序运行过程中触发了的竞态条件才能被竞态条件检测工具检测出来。

### 典型的竞态条件

```
func main() {
	var wg sync.WaitGroup
	wg.Add(5)
	for i := 0; i < 5; i++ {
		go func() {
			fmt.Println(i) // Not the 'i' you are looking for.
			wg.Done()
		}()
	}
	wg.Wait()
}

//===================================================

func main() {
    start := time.Now()
    reset := make(chan bool)
    var t *time.Timer
    t = time.AfterFunc(randomDuration(), func() {
        fmt.Println(time.Now().Sub(start))
        reset <- true
    })
    for time.Since(start) < 5*time.Second {
        <-reset
        t.Reset(randomDuration())
    }
}
```

### 阅读资料
- [https://blog.golang.org/race-detector](https://blog.golang.org/race-detector)
- [https://golang.org/doc/articles/race_detector.html](https://golang.org/doc/articles/race_detector.html)


