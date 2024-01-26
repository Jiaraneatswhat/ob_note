# 1 读取数据
- PyTorch 中的加载数据流程
	- 加载数据，提取出特征和标签，封装成 `tensor`
	- 创建一个 `DataSet` 对象
	- 创建一个 `DataLoader` 对象，用于实现数据加载的方式
	- 通过 `enumerate` 迭代 `DataLoader`，调用其 `__iter__ `方法加载数据
## 1.1 DataSet
- `torch.utils.data.dataset.py` 中定义了 `DataSet` 类：
```python
class Dataset(Generic[T_co]):
	# 定义的 __getitem__ 方法需要子类实现
	def __getitem__(self, index) -> T_co:  
	    raise NotImplementedError("Subclasses of Dataset should implement __getitem__.")
	# 重载了 + 运算符，两个 DataSet 对象求和后返回一个 ConcatDataset 对象
	def __add__(self, other: 'Dataset[T_co]') -> 'ConcatDataset[T_co]':  
    return ConcatDataset([self, other])
```
- `ConcatDataset` 也定义在 `torch.utils.data.dataset` 中：
```python
def __init__(self, datasets: Iterable[Dataset]) -> None:  
    super().__init__()  
    # 会将求和的两个 DataSet 对象放在 list 中
    self.datasets = list(datasets)   
    self.cumulative_sizes = self.cumsum(self.datasets)
```
- 自定义一个 `DataSet` 的子类：
```python
class MyData(Dataset):  
      
    def __init__(self, root_dir, label_dir):  
        self.root_dir = root_dir  
        self.label_dir = label_dir  
        self.path = os.path.join(self.root_dir, self.label_dir)  
        # 将图像地址封装在 list 中，后续可以通过 __getitem__ 直接索引
        self.img_path = os.listdir(self.path)  
        
    def __getitem__(self, index):  
        img_name =  self.img_path[index]  
        img_item_path = os.path.join(self.root_dir, self.label_dir, img_name)  
        img = Image.open(img_item_path)  
        label = self.label_dir  
        return img, label  
      
    def __len__(self):  
        return len(self.img_path)
```
## 1.2 DataLoader
-  `torch.utils.data.dataloader.py` 中定义了 `DataLoader` 类：
```python
class DataLoader(Generic[T_co]):
	dataset: Dataset[T_co]  
	batch_size: Optional[int] # 默认 1  
	num_workers: int # 线程数 0 -> main 线程
	pin_memory: bool # 内存寄存, 默认 False, 表示在数据返回前，是否复制到 CUDA 内存中
	drop_last: bool # 样本数不能被 batch_size 整除时，舍弃最后一批数据
	timeout: float # 读数据超时时间 

	def __init__(self, 
		dataset: Dataset[T_co], 
		batch_size: Optional[int] = 1,  
		shuffle: Optional[bool] = None, # 打乱每个 epoch 的数据
		sampler: Union[Sampler, Iterable, None] = None,  
		batch_sampler: Union[Sampler[List], Iterable[List], None] = None, 
        num_workers: int = 0, collate_fn: Optional[_collate_fn_t] = None,  
		pin_memory: bool = False, 
		drop_last: bool = False,  
        timeout: float = 0, 
		pin_memory_device: str = ""):
		...
```
### 1.2.1 读取数据逻辑
- 将一个 `batch` 的数据进行合并
```python
# DataLoader 在迭代时使用 _SingleProcessDataLoaderIter, 继承自 _BaseDataLoaderIter
class _SingleProcessDataLoaderIter(_BaseDataLoaderIter):  
    def __init__(self, loader):  
        super().__init__(loader)  
		## 根据 _DatasetKind 定义一个 map 的 fetcher 或 iter 的 fetcher
        self._dataset_fetcher = _DatasetKind.create_fetcher(  
            self._dataset_kind, self._dataset, self._auto_collation, self._collate_fn, self._drop_last)  
  
    def _next_data(self):  
        index = self._next_index()  # may raise StopIteration  
        # 通过 fetcher 读取数据
        data = self._dataset_fetcher.fetch(index)  # may raise StopIteration  
        if self._pin_memory:  
            data = _utils.pin_memory.pin_memory(data, self._pin_memory_device)  
        return data

# fetch.py
class _IterableDatasetFetcher(_BaseDatasetFetcher):  
    def __init__(self, dataset, auto_collation, collate_fn, drop_last):  
        super().__init__(dataset, auto_collation, collate_fn, drop_last)  
        self.dataset_iter = iter(dataset)  
        self.ended = False  
  
    def fetch(self, possibly_batched_index):  
        if self.ended:  
            raise StopIteration  
  
        if self.auto_collation:  
            data = []  
            for _ in possibly_batched_index:  
                try:  
		            # 自动通过 append 方法合并
                    data.append(next(self.dataset_iter))  
                except StopIteration:  
                    self.ended = True  
                    break            
        else:  
            data = next(self.dataset_iter)  
        # 默认的 collate_fn 将 data 转为 tensor
        return self.collate_fn(data)
```
### 1.2.2 shuffle
- `shuffle` 参数用于打乱数据，使数据更具有独立性，一般用于训练集
```python

```