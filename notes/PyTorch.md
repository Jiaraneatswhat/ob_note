# 1 读取数据
- PyTorch 中的加载数据流程
	- 加载数据，提取出特征和标签，封装成 tensor
	- 创建一个 DataSet 对象
	- 创建一个 DataLoader 对象，用于实现数据加载的方式
	- 循环 DataLoader，训练数据
## 1.1 DataSet
- `torch.utils.data.dataset` 中定义了 `DataSet` 类：
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