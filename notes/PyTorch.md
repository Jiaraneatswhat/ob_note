# 1 读取数据
- PyTorch 中的加载数据流程
	- 加载数据，提取出特征和标签，封装成 tensor
	- 创建一个 DataSet 对象
	- 创建一个 DataLoader 对象，用于实现数据加载的方式
	- 循环 DataLoader，训练数据
## 1.1 DataSet
- `torch.utils.data.dataset` 中定义了 DataSet 类：
```python
class Dataset(Generic[T_co]):
	# 定义的 __getitem__ 方法需要子类实现
	def __getitem__(self, index) -> T_co:  
	    raise NotImplementedError("Subclasses of Dataset should implement __getitem__.")

	def __add__(self, other: 'Dataset[T_co]') -> 'ConcatDataset[T_co]':  
    return ConcatDataset([self, other])
```