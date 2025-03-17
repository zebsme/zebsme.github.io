+++

title = "实现简单的关系型数据库"
date = 2025-03-10
authors = ["zeb"]

+++

Datafusion

Join的实现

## type system

use apache arrow as the basis of our type system

## logical Plan

serialization 

- json 等 

- 与语言无关的序列化格式 Avro Thrift Protocol Buffers

- substrait 关系代数的统一标准

// 初始化每个分区的 BooleanBufferBuilder
                    let num_rows = batch.num_rows();
                    let mut builders: Vec<BooleanBufferBuilder> = (0..*partitions)
                        .map(|_| {
                            let mut builder = BooleanBufferBuilder::new(num_rows);
                            builder.append_n(num_rows, false); // 初始填充 false
                            builder
                        })
                        .collect();

```rust
                // 单次循环设置所有分区的掩码
                for (idx, &hash) in hash_buffer.iter().enumerate() {
                    let partition = (hash % *partitions as u64) as usize;
                    builders[partition].set_bit(idx, true); // 直接设置位值
                }

                // 转换为 BooleanArray
                let masks: Vec<BooleanArray> = builders
                    .into_iter()
                    .map(|mut builder| {
                        BooleanArray::from(builder.finish())
                    })
                    .collect();
                timer.done();

                let partitioner_timer = &self.timer;
                // 使用 filter 生成分区数据
                let it = masks
                    .into_iter()
                    .enumerate()
                    .filter(|(_, mask)| mask.true_count() > 0)
                    .map(move |(partition, mask)| {
                        let _timer = partitioner_timer.timer();
                        let columns = batch.columns()
                            .iter()
                            .map(|col| filter(col, &mask).map_err(|e| DataFusionError::ArrowError(e, None)))
                            .collect::<Result<Vec<ArrayRef>>>()?;
                        RecordBatch::try_new(batch.schema(), columns)
                            .map(|batch| (partition, batch)).map_err(|e| DataFusionError::ArrowError(e, None))
                    });

                Box::new(it)
```

// 2. 预分配列式缓冲区 (替代位掩码)
                    let mut partition_buffers: Vec<Vec<Box<dyn ArrayBuilder>>> = (0..*partitions)
                        .map(|_| {
                            batch.schema().fields().iter()
                                .map(|field| make_builder(field.data_type(), batch.num_rows()))
                                .collect::<Result<Vec<_>>>()
                        })
                        .collect::<Result<Vec<_>>>()?;

                    // 3. 单次遍历填充数据
                    for (row_idx, &hash) in hash_buffer.iter().enumerate() {
                        let partition = (hash % *partitions as u64) as usize;
                        for (col_idx, array) in batch.columns().iter().enumerate() {
                            let builder = &mut partition_buffers[partition][col_idx];
                            append_value(array, row_idx, builder)?;
                        }
                    }
    
                    // Finished building index-arrays for output partitions
                    timer.done();
    
                    // Borrowing partitioner timer to prevent moving `self` to closure
                    let partitioner_timer = &self.timer;
                    // let it = indices
                    //     .into_iter()
                    //     .enumerate()
                    //     .filter_map(|(partition, indices)| {
                    //         let indices: PrimitiveArray<UInt32Type> = indices.into();
                    //         (!indices.is_empty()).then_some((partition, indices))
                    //     })
                    //     .map(move |(partition, indices)| {
                    //         // Tracking time required for repartitioned batches construction
                    //         let _timer = partitioner_timer.timer();
                    //
                    //         // Produce batches based on indices
                    //         let columns = take_arrays(batch.columns(), &indices, None)?;
                    //
                    //         let mut options = RecordBatchOptions::new();
                    //         options = options.with_row_count(Some(indices.len()));
                    //         let batch = RecordBatch::try_new_with_options(
                    //             batch.schema(),
                    //             columns,
                    //             &options,
                    //         )
                    //         .unwrap();
                    //
                    //         Ok((partition, batch))
                    //     });
                    let schema = batch.schema();
                    let batches = partition_buffers
                        .into_iter()
                        .enumerate()
                        .filter(|(_, cols)| cols.iter().any(|c| c.len() > 0))
                        .map(move|(partition, cols)| {
                            let _timer = partitioner_timer.timer();
                            let arrays = cols.into_iter()
                                .map(|mut b| b.finish())
                                .collect::<Vec<ArrayRef>>();
                            RecordBatch::try_new(schema.clone(), arrays)
                                .map(|batch| (partition, batch))
                                .map_err(|e| DataFusionError::ArrowError(e, None))
                        });
    
                    Box::new(batches)
                }
            };
        Ok(it)
    }
    
    // return the number of output partitions
    fn num_partitions(&self) -> usize {
        match self.state {
            BatchPartitionerState::RoundRobin { num_partitions, .. } => num_partitions,
            BatchPartitionerState::Hash { num_partitions, .. } => num_partitions,
        }
    }
}
// 修改 make_builder 函数以支持 Utf8View 类型
fn make_builder(data_type: &DataType, capacity: usize) -> Result<Box<dyn ArrayBuilder>, DataFusionError> {
    match data_type {
        DataType::Int32 => Ok(Box::new(arrow::array::Int32Builder::with_capacity(capacity))),
        DataType::Utf8 => Ok(Box::new(arrow::array::StringBuilder::with_capacity(capacity, capacity * 8))),
        DataType::LargeUtf8 => Ok(Box::new(arrow::array::LargeStringBuilder::with_capacity(capacity, capacity * 8))),
        DataType::Utf8View => {
            // 需要 arrow 13.0.0 或更高版本
            Ok(Box::new(arrow::array::StringViewBuilder::with_capacity(capacity)))
        }
        _ => Err(DataFusionError::NotImplemented(format!("Unsupported type: {:?}", data_type))),
    }
}

// 修改 append_value 函数以支持 Utf8View 类型
fn append_value(array: &ArrayRef, index: usize, builder: &mut Box<dyn ArrayBuilder>) -> Result<()> {
    use arrow::array::*;
    match array.data_type() {
        DataType::Int32 => {
            let builder = builder.as_any_mut().downcast_mut::<Int32Builder>().unwrap();
            let value = array.as_any().downcast_ref::<Int32Array>().unwrap().value(index);
            builder.append_value(value)
        }
        DataType::Utf8 => {
            let builder = builder.as_any_mut().downcast_mut::<StringBuilder>().unwrap();
            let value = array.as_any().downcast_ref::<StringArray>().unwrap().value(index);
            builder.append_value(value)
        }
        DataType::LargeUtf8 => {
            let builder = builder.as_any_mut().downcast_mut::<LargeStringBuilder>().unwrap();
            let value = array.as_any().downcast_ref::<LargeStringArray>().unwrap().value(index);
            builder.append_value(value)
        }
        DataType::Utf8View => {
            let builder = builder.as_any_mut().downcast_mut::<StringViewBuilder>().unwrap();
            let view = array.as_any().downcast_ref::<StringViewArray>().unwrap().value(index);
            builder.append_value(view)
        }
        _ => return Err(DataFusionError::NotImplemented("Type not supported".into())),
    }
    Ok(())
