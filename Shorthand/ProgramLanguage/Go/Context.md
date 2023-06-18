# go context是什么？
go标准库context提供的context.Context接口可以在go中来设置截止日期、同步信号、传递相关请求值等。这个标准库有此接口的众多实现。

go起的后端服务对于一个请求可能会启动几个goroutine同时工作。在goroutine构成的树形结构中对信号进行同步以减少计算资源的浪费是`context.Context`的最大作用。