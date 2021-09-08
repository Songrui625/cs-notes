# Query Execution
在实现之前，我们需要对Query Execution有个总体认识。Query Execution位于DBMS整体架构的第三层，如下图，Query Execution两个重要组件是`Query Executor`和`Query Plan`。

![image-20210815181844655](.\figure\image-20210815181057898.png)



 当我们需要执行一条SQL语句(select, update, insert, delete)时，DBMS会为每一条SQL语句生成一个`executor`和一系列的`query plan`，然后`executor`负责执行这些`query plan`。

`Executor`需要支持以下操作：

* `Access Method`：顺序扫描，索引扫描
* `Modification`：插入、更新、删除
* `各种各样`：嵌套循环联接(`nested loop join`)，索引嵌套循环联接(`nested loop join`)，聚集(`aggregation`)，限制(`Limit`)/偏移(`Offset`)。

### Query Plan采用哪种模型实现？

`迭代器模型`，也叫`🌋火山模型`或者`流水线模型`, 我们需要实现迭代器模型的`Next`函数，每次迭代会执行一次`Next`函数时，`Next`函数会返回一个tuple并发送到父节点，直到`Next`返回一个`nullptr`迭代结束, 在这里我们采用的是一种自底向上的视图，在实际实现中，我们需要自顶向下调用`Next`，及在父节点调用`Next`函数，然后会去调用孩子节点的`Next`函数，直到叶节点。

## Catalog

**catalog**负责维护数据库所有的元数据，例如有哪些`table`、`index`等等，例如当我们创建了一个新表，我们会在`catalog`中添加关于这个新表的元数据，哪个事务执行的操作、表的名字是什么、表的schema。

我们来看下`Table`的metadata和`Index`的Metadata，都很好理解，值得一提的是，我们是可以直接通过Metadata拿到`Table`和`Index`，因为Metadata中存储了一个智能指针`unique_ptr`指向`Table`和`Index`对象。

```c++
/**
 * Metadata about a table.
 */
struct TableMetadata {
  TableMetadata(Schema schema, std::string name, std::unique_ptr<TableHeap> &&table, table_oid_t oid)
      : schema_(std::move(schema)), name_(std::move(name)), table_(std::move(table)), oid_(oid) {}
  Schema schema_;
  std::string name_;
  std::unique_ptr<TableHeap> table_;
  table_oid_t oid_;
};

/**
 * Metadata about a index
 */
struct IndexInfo {
  IndexInfo(Schema key_schema, std::string name, std::unique_ptr<Index> &&index, index_oid_t index_oid,
            std::string table_name, size_t key_size)
      : key_schema_(std::move(key_schema)),
        name_(std::move(name)),
        index_(std::move(index)),
        index_oid_(index_oid),
        table_name_(std::move(table_name)),
        key_size_(key_size) {}
  Schema key_schema_;
  std::string name_;
  std::unique_ptr<Index> index_;
  index_oid_t index_oid_;
  std::string table_name_;
  const size_t key_size_;
};
```

我们再来看一下Catalog维护哪些成员变量，`next_table_oid_`和`next_index_oid_`的类型都为`std::atomic`，这是保证事务并发的创建表、索引时能保证这两个变量不会`lost update`。

```c++
 private:
  [[maybe_unused]] BufferPoolManager *bpm_;
  [[maybe_unused]] LockManager *lock_manager_;
  [[maybe_unused]] LogManager *log_manager_;

  /** tables_ : table identifiers -> table metadata. Note that tables_ owns all table metadata. */
  std::unordered_map<table_oid_t, std::unique_ptr<TableMetadata>> tables_;
  /** names_ : table names -> table identifiers */
  std::unordered_map<std::string, table_oid_t> names_;
  /** The next table identifier to be used. */
  std::atomic<table_oid_t> next_table_oid_{0};
  /** indexes_: index identifiers -> index metadata. Note that indexes_ owns all index metadata */
  std::unordered_map<index_oid_t, std::unique_ptr<IndexInfo>> indexes_;
  /** index_names_: table name -> index names -> index identifiers */
  std::unordered_map<std::string, std::unordered_map<std::string, index_oid_t>> index_names_;
  /** The next index identifier to be used */
  std::atomic<index_oid_t> next_index_oid_{0};
```



## Executors

每个`Executor`需要实现两个接口`Init`和`Next`，前面介绍过了`Next`，不再赘述。`Init`负责初始化一些`Executor`在`Next`中需要使用的对象，例如`Sequential Scan`中可能需要调用`TableHeap`的迭代器迭代整个table每一个tuple，所以我们得维护需要遍历的`TableHeap`的对象。

由于Bustub还不支持SQL，所以我们得手写`Query Plan`，但是有了`Query Plan`如何构造对应的`Query Executor`？Bustub中采用了工厂模式来根据`Query Plan`的类型来构造对应类型的`Executor`并返回。可以看到代码中存在`dynamic_cast`，所以不是在编译期完成的，而是在运行时进行类型检查并构造相应的`Executor`，需要说明的是`dynamic_cast`仅适用于指针或者引用。

```c++

std::unique_ptr<AbstractExecutor> ExecutorFactory::CreateExecutor(ExecutorContext *exec_ctx,
                                                                  const AbstractPlanNode *plan) {
  switch (plan->GetType()) {
    // Create a new sequential scan executor.
    case PlanType::SeqScan: {
      return std::make_unique<SeqScanExecutor>(exec_ctx, dynamic_cast<const SeqScanPlanNode *>(plan));
    }

    case PlanType::IndexScan: {
      return std::make_unique<IndexScanExecutor>(exec_ctx, dynamic_cast<const IndexScanPlanNode *>(plan));
    }

    // Create a new insert executor.
    case PlanType::Insert: {
      auto insert_plan = dynamic_cast<const InsertPlanNode *>(plan);
      auto child_executor =
          insert_plan->IsRawInsert() ? nullptr : ExecutorFactory::CreateExecutor(exec_ctx, insert_plan->GetChildPlan());
      return std::make_unique<InsertExecutor>(exec_ctx, insert_plan, std::move(child_executor));
    }
	...
    default: {
      BUSTUB_ASSERT(false, "Unsupported plan type.");
    }
  }
}
```

而真正调用`ExecutorFactory`获得`Query Executor`的是`ExecutionEngine`，`ExecutionEngine`主要就做一件事，根据`Query Plan`构造`Query Executor`，执行`Executor`，实现如下。`Execute`参数传了一个指向`vector`的指针，这是因为有些`query plan`，我们并不需要检索`tuple`，例如`insert`,`update`。代码中还涉及了一个`ExecutorContext`类型的参数，从类名就能看出来，这个类型维护了执行此次`query plan`的上下文，例如要对哪个`table`，或者哪个`index`进行操作等等。

```c++

  bool Execute(const AbstractPlanNode *plan, std::vector<Tuple> *result_set, Transaction *txn,
               ExecutorContext *exec_ctx) {
    // construct executor
    auto executor = ExecutorFactory::CreateExecutor(exec_ctx, plan);

    // prepare
    executor->Init();

    // execute
    try {
      Tuple tuple;
      RID rid;
      while (executor->Next(&tuple, &rid)) {
        if (result_set != nullptr) {
          result_set->push_back(tuple);
        }
      }
    } catch (Exception &e) {
      // TODO(student): handle exceptions
    }

    return true;
  }
```







