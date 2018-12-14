# Gitbook

使用 Gitbook 方便文档查阅，文档更有组织。

## Getting started

安装 gitbook  
```
npm install gitbook-cli -g
```

初始化
```
gitbook init
``` 
会创建 `SUMMARY.md` 和 `README.md` 文档组织管理文件。

编辑完成后
```
gitbook build
```
创建静态网页，gitbook查看到的就是静态网页

## 文档托管

这里包括两个概念，文档 repo 和 gitbook 显示

#### Gitbook.io 托管

优点：
- gitbook 官方支持
- 自带 `gitbook editor`，在线编辑器
- 支持链接 github 仓库，同步更新文档

缺点：
 - 服务器响应慢，操作太差
 - editor legacy, 登录不了，无法同步网站
 - 个人只能添加一个 Space（类似repo）

#### Github

优点：
- 常用熟悉
- 好用，速度快

缺点：
- github pages 不知道怎么用。。。有空再看

#### Gitlab

优点：
- 熟悉
- 支持 CI

缺点：
- gitlab pages 暂时不会用
- repo 都在 github，只有文档在 gitbook 不太好


#### 所以目前方案

- github 托管文件
- gitbook.io 显示文档
- 直接使用 markdown  / gitbook editor 编辑