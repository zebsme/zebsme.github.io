+++
title = "Deep Dive into Apache Arrow"
date = 2025-02-20
authors = ["zeb"] 

+++



当谈及文件格式时，可以考虑以下三个方面：

1. Size
2. Serialize/Deserialize Speed
3. Ease of use

The primary goal of datafusion is become 通用 format of data processing and analtics. Systems have their own data formats, having to be copied and converted during runtime. And in many cases, the serialization and deserialization process can take up for about 90% of the processing time.

## Arrow Flight



为什么使用列式格式

There is often many debate about whether a database shoud be row-oriented or column-oriented

Most operating systems, while reading data into main memory and CPU caches, will attempt to make predictions about what memory it is going to need next. In our example table of archers, consider how many memory pages (refer to [*Chapter 3*](javascript:void(0)) for more details) of data would have to be accessible and traversed to get a list of unique locations if the data were organized in row or column orientations:
大多数作系统在将数据读入主内存和 CPU 缓存时，会尝试预测接下来需要什么内存。 在我们的 archers 示例表中，考虑如果数据按行或列方向组织，则必须访问和遍历多少个数据的内存页（有关详细信息，请参阅[*第 3 章*](javascript:void(0))）才能获得唯一位置列表

As shown in *Figure 1**.4*, the columnar format keeps the data organized by column instead of by row. As a result, operations such as grouping, filtering, or aggregations based on column values become much more efficient to perform since the entire column is already contiguous in memory. Considering memory pages again, it’s plain to see that for a large table, there would be significantly more pages that need to be traversed to get a list of unique locations from a row-oriented buffer than a columnar one. Fewer page faults and more cache hits mean increased performance and a happier CPU. Computational routines and query engines tend to operate on subsets of the columns for a dataset rather than needing every column for a given computation, making it significantly more efficient to operate on columnar data.
如图 *1.4* 所示，列式格式保持按列而不是按行组织数据。 因此，基于列值的分组、筛选或聚合等作的执行效率要高得多，因为整个列在内存中已经是连续的。 再次考虑内存页，很明显，对于大型表，需要遍历的页数要从面向行的缓冲区中获取唯一位置列表的页数要比列式缓冲区多得多。 更少的页面错误和更多的缓存命中意味着更高的性能和更满意的 CPU。 计算例程和查询引擎倾向于对数据集的列子集进行作，而不是需要每一列进行给定的计算，这使得对列式数据进行作的效率明显更高。

By keeping the column data contiguous in memory, it allows the computations to be vectorized. Most modern processors have **single instruction, multiple data** (**SIMD**) instructions available that can be taken advantage of for speeding up computations and require the data to be in a contiguous block of memory so that they can operate on it. This concept can be found heavily utilized by graphics cards, and Arrow provides libraries to take advantage of **graphics processing units** (**GPUs**) precisely because of this. Consider an example where you might want to multiply every element of a list by a static value, such as performing a currency conversion on a column of prices while using an exchange rate:
通过将列数据在内存中保持连续，它允许对计算进行矢量化。 大多数现代处理器都有可用的**单指令多数据** （**SIMD**） 指令，可以利用它来加速计算，并要求数据位于连续的内存块中，以便它们可以对其进行作。 这个概念可以被显卡大量使用，正是因为这个原因，Arrow 提供了库来利用**图形处理 units** （**GPU**）。考虑一个示例，您可能希望将列表的每个元素乘以静态值，例如，在对 prices 列执行货币转换，同时使用汇率：

![B21920_01_05](E:/Obsidian/Go/images/B21920_01_05.jpg)
