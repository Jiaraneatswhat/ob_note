# 1 读取数据
- PyTorch 中的加载数据流程
	- 加载数据，提取出特征和标签，封装成 tensor
	- 创建一个 DataSet 对象
	- 创建一个 DataLoader 对象，用于实现数据加载的方式
	- 循环 DataLoader，训练数据
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
	