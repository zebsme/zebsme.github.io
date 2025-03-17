+++
title = "From ImageStore to Qemu-nbd"
date = 2023-05-20
authors = ["zeb"] 

+++

# 镜像仓库对接 nbd

## 背景 

直接把镜像仓库中维护的分层、切片镜像以 nbd 方式（只读）暴露出来，可以大大提高如下体
验：

1. 从镜像仓库导出镜像到 Ceph 性能预期可以提高 N 倍（N > 5)；
2. 灾备模块也可以实现快速拉起；
3. 镜像仓库导出镜像时间和空间都得到提升（不再需要临时文件）。

## 可行的方案 

### 修改 qemu-nbd 只读暴露镜像 

修改 qemu/qemu-nbd, 增加新的 BlockDriver 直接读取镜像仓库中存放的分层、切片镜像。需
要处理的逻辑包括：

1. 通过镜像仓库的 image manifest 文件来处理 backing chain 逻辑；
2. 通过镜像仓库的 blob manifest 文件来处理 chunk 读取逻辑；

假想示例，对于镜像 zstore://foo/bar：
$ qemu-nbd -f imf2 -p 10810 foo``/bar
好处：最好的性能，且 qemu/qemu-nbd/qemu-io 都可以利用。

### 通过 FUSE 只读暴露镜像 

通过 FUSE 把镜像只读暴露成 qcow2 backing chain, 这样就不需要再修改 qemu/qemu-nbd. 
只是把上面需要修改的逻辑完全放在了 FUSE 应用里。

假想示例，对于镜像 zstore://foo/bar：

$ chunkfs -conf /data/zstore/zstore``.yaml  -dst /tmp/foo/ -img foo``/bar
 $ ls /tmp/foo
 6f0a5968aa249a.qcow2 bdfcb51fd7aa00.qcow2 f233c4f119343.qcow2 
$ qemu-nbd -r -f qcow2 -p 10810 /tmp/foo/6f0a5968aa249a.qcow2 
好处：影响面较小，足够好的性能，且可以给镜像仓库的导出功能复用（快、且省空间）。

## 实现 

基于 golang 第三方库 github.com/jacobsa/fuse, 这样可以重用现有镜像仓库解析 imf2 文件以
及寻址 chunk 的代码。

###  dentry 的构造 

首先需要循环读取 image manifest, 生成 dentry 记录。这样，opendir() 后读取到的是当前镜像
对应的 backing chain 列表。文件名为 image-id.qcow2, 对于 foo/bar，这里 'foo' 是 
namespace, 'bar' 是 image-id. 注意，虽然文件名后缀是 qcow2，但本身可能是 raw/iso 之类
（base image)，本算法因为下面红框内的处理，依然生效。

### 文件 I/O 的处理 

针对文件的读，计算 read 的 offset/range，映射成对  chunk 的读写再返回。如果 offset/range 
在当前 qcow2 header 的 backing file 指向的范围，则处理相应的内容，填充为 parent 的 
"image-id.qcow2".
针对文件的写，直接报错 EPERM（不允许写）。
type chunkFS struct {
 fuseutil.NotImplementedFileSystem
 }
 func (fs *chunkFS) ReadFile(
 ctx context.Context,
 op *fuseops.ReadFileOp) error {
 }
 cfg := &fuse.MountConfig{
 ReadOnly: true``,
 FSName:   "zstore"``,
 Subtype:  "zstore"``,
}

###  Partial Read 

Qcow2 文件的 cluster size 一般设置为 64KB 到 1MB 之间。镜像仓库的切片大小有两种：4MB 
或者 64MB. 理论上，如果按 cluster 读取，一个 read() 调用不跨 cluster 的情况下，每次读取都
会在一个切片里。但实现在，用 FUSE 处理读的时候，也处理了跨切片的情况。解决方法也很简
单：每次只会读一个切片内的数据，并返回读取的字节数。因为 read() 的定义是允许 partial 返
回的，因此这种情况下，上层调用者会自动再次读取下一个切片的内容。

### 自动修正 backing file 

要让 qcow2 文件的 backing file 自动指向其在 FUSE 挂载点中的 parent, 代码需要解析 qcow2 
header 中指向的 backing file offset, 并在读取相应的内容后，替换其中的内容为 parent qcow2 
文件路径。另外还需要处理 qcow2 header 中 backing file size 字段的值。根据 qcow2 格式定
义，backing file name 会保存在第一个 cluster（且不会跨 cluster）.

## 已知问题 

CentOS Stream 8 用 Go 1.18.5 编译的 FUSE 程序在 CentOS 7.6 (kernel 3.10.0-957) 上运行虽
然不会报错，但挂载的目录无法访问：ls 报错 ECONNREFUSED 

做的努力

手动测试 

## 数据面验证 

镜像仓库目前具有镜像导出功能：重新把分层、发片镜像拼装回去之后，再压缩合并为单个镜像文
件。这目前也是导出到 Ceph 的一个中间步骤（因此比较耗时、耗 CPU、耗 I/O）。
测试用例可以分别用新、老方式导出镜像，然后再用 qemu-img compare 对镜像数据做对比。

## 展望 

实现将镜像仓库分层、切片镜像只读暴露成 nbd 的方法（保证镜像/备份数据只读的语义）。

这样可以做到：
备份冷恢复利用 qemu-io 可以快速恢复到任意存储；
灾备数据恢复到 Ceph 也可以通过 qemu-io 快速导镜像。