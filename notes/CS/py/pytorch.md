# 1 Tensor
- `tensor` 是标量，向量，矩阵的高维扩展
- `numpy` 不能在 `gpu` 上加速，`tensor` 可以
- tensor 的属性
	- data
	- grad
	- grad_fn：创建 tensor 使用的函数
	- requires_grad：是否需要梯度
	- is_leaf：是否为计算图中的叶节点
	- dtype
	- shape
	- device
- `tensor` 在物理上连续存储，通过 `stride` 转换索引
- `c10::TensorImpl` 表示数据的逻辑操作，实际的操作在 `c10::StorageImpl` 中进行
## 1.1 创建 tensor
```python
# 直接创建 tensor
torch.tensor(data: Any, dtype: Optional[_dtype] = None, device: Device = None, requires_grad: _bool = False)

# 创建未初始化的 tensor
torch.empty(_*size_, _*_, _out=None_, _dtype=None_, _layout=torch.strided_, _device=None_, _requires_grad=False_, _pin_memory=False_, _memory_format=torch.contiguous_format_) → Tensor

# 创建均匀分布的 tensor
torch.rand(_*size_, _*_, _generator=None_, _out=None_, _dtype=None_, _layout=torch.strided_, _device=None_, _requires_grad=False_, _pin_memory=False_) → Tensor

# 创建全 0 和全 1 的 tensor
torch.zeros(_*size_, _*_, _out=None_, _dtype=None_, _layout=torch.strided_, _device=None_, _requires_grad=False_) → Tensor

torch.ones(_*size_, _*_, _out=None_, _dtype=None_, _layout=torch.strided_, _device=None_, _requires_grad=False_) → Tensor

# 创建高斯分布的 tensor
torch.randn(_*size_, _*_, _generator=None_, _out=None_, _dtype=None_, _layout=torch.strided_, _device=None_, _requires_grad=False_, _pin_memory=False_) → Tensor

# 和 numpy 转换
torch.from_numpy()
torch.Tensor()
```
## 1.2 Tensor 属性
```python
# 存储数据类型
tensor.dtype

# shape 和维度
tensor.shape
tensor.ndim

# 存储设备
tensor.device
tensor.is_cuda

# 梯度
tensor.grad
```
## 1.3 常用方法
### 1.3.1 squeeze()
```python
# torch.squeeze(_input_, _dim=None_) → Tensor
# 移除 dim = 1 的维度，如果指定了 dim，指定 dim 为 1 时才会移除
torch.squeeze(torch.empty((2, 1, 2, 1))).shape = (2, 2)
torch.squeeze(torch.empty((2, 1, 2, 1)), dim=0).shape = (2, 1, 2, 1)
torch.squeeze(torch.empty((2, 1, 2, 1)), dim=1).shape = (2, 2, 1)
```
### 1.3.2 permute()
```python
# torch.permute(_input_, _dims_) → Tensor
# dim 指定维度的顺序
torch.permute(torch.empty((2, 4, 6)), dims=(1, 0, 2)).shape = (4, 2, 6)
```
### 1.3.3 expand()
```python
# 将维度为 1 的 dim 扩展至其他维
# -1 表示不扩展
# 扩展的 tensor 不会分配新的内存，只在原来的基础上创建新的视图
torch.empty((1, 3, 6)).expand(2, -1, -1).shape = (2, 3, 6)
torch.empty((1, 1, 6)).expand(2, 3, -1).shape = (2, 3, 6)
```
### 1.3.4 repeat()
```python
# Tensor.repeat(_*sizes_) → Tensor
# 在指定维度复制数据
torch.tensor([1, 2]).repeat((2, 1)) = [[1, 2], [1, 2]]
torch.tensor([1, 2]).repeat((2, 2, 1)) = [[[1, 2], [1, 2]],
									 [[1, 2], [1, 2]]]
```
## 1.4 运算
```python
a + b
torch.add_(a, b) 等价于 a += b，会改变 a

a - b
torch.sub(a, b)

a * b
torch.mul(a, b)

a / b
torch.div(a, b)
```
## 1.5 自动求导
- 可以用 `torch.autograd.backward()` 来自动计算变量的梯度
	- 定义 `tensor` 时加上 `requires_grad=True` 表示需要求偏导
```python
x = torch.randn(3, requires_grad=True)  
y = x + 2  
z = y * y * 3   
z = z.mean()  
z.backward()  
x.grad = 2 * (x + 2)
```
- 停止计算梯度
	- `tensor.requires_grad_(False)`
	- `可求导的 tensor.detach()` 会返回一个内容相同，不能计算梯度的新 `tensor`
	- `with torch.no_grad():` 作用域中定义的都是不需要求导的 `tensor`
- `tensor.grad.zero_()` 可以清空梯度
# 2 读取数据
- PyTorch 中的加载数据流程
	- 加载数据，提取出特征和标签，封装成 `tensor`
	- 创建一个 `DataSet` 对象
	- 创建一个 `DataLoader` 对象，用于实现数据加载的方式
	- 通过 `enumerate` 迭代 `DataLoader`，调用其 `__iter__ `方法加载数据
## 2.1 DataSet
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
## 2.2 DataLoader
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
### 2.2.1 读取数据逻辑
- 将一个 `batch` 的数据进行合并
```python
# DataLoader 在迭代时使用 _SingleProcessDataLoaderIter, 继承自 _BaseDataLoaderIter
# enumerate 进入 _get_iterator() 方法，返回一个 _SingleProcessDataLoaderIter
def _get_iterator(self) -> '_BaseDataLoaderIter':  
    if self.num_workers == 0:  
        return _SingleProcessDataLoaderIter(self)  
    else:  
        self.check_worker_number_rationality()  
        return _MultiProcessingDataLoaderIter(self)
        
class _SingleProcessDataLoaderIter(_BaseDataLoaderIter):  
    def __init__(self, loader):  
        super().__init__(loader)  
		## 根据 _DatasetKind 定义一个 map 的 fetcher 或 iter 的 fetcher
        self._dataset_fetcher = _DatasetKind.create_fetcher(  
            self._dataset_kind, self._dataset, self._auto_collation, self._collate_fn, self._drop_last)  
  
    def _next_data(self):  
        index = self._next_index()  # may raise StopIteration  
        # 通过 fetcher 传入 index 读取数据
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
### 2.2.2 shuffle
- `shuffle` 参数用于打乱数据，使数据更具有独立性，一般用于训练集
```python
# 通过 sampler 来实现 shuffle
if sampler is None:  # give default samplers  
    if self._dataset_kind == _DatasetKind.Iterable:  
        # See NOTE [ Custom Samplers and IterableDataset ]  
        sampler = _InfiniteConstantSampler()  
    else:  # map-style  
        if shuffle:  
            sampler = RandomSampler(dataset, generator=generator)  # type: ignore[arg-type]  
        else:  
            sampler = SequentialSampler(dataset)  # type: ignore[arg-type]  
  
if batch_size is not None and batch_sampler is None:  
    # auto_collation without custom batch_sampler  
    batch_sampler = BatchSampler(sampler, batch_size, drop_last)
```
# 3 transforms
- `torchvision.transforms` 主要定义了一些与图像增广相关的操作
- 图像常见的打开方式
	- `PIL -> Image.open()`
	- `tensor -> ToTensor()`
	- `ndarray -> cv2.imread()`
## 3.1 Compose
```python
class Compose:
	def __init__(self, transforms):  
    # 将传入的 transforms 列表组合在一起
    self.transforms = transforms

	def __call__(self, img):  
	# 循环调用
    for t in self.transforms:  
        img = t(img)  
    return img
```
## 3.2 Normalize
- `output[channel] = (input[channel] - mean[channel]) / std[channel]`
```python
class Normalize(torch.nn.Module):
	def __init__(self, mean, std, inplace=False):  
	    super().__init__()  
	    self.mean = mean  
	    self.std = std  
	    self.inplace = inplace

	def forward(self, tensor: Tensor) -> Tensor:
		# 调用 torch.functional 实现标准化
		return F.normalize(tensor, self.mean, self.std, self.inplace)
```
## 3.3 Resize
```python
class Resize(torch.nn.Module):
	# 默认双线性插值
	def __init__(self, size, interpolation=InterpolationMode.BILINEAR, max_size=None, antialias="warn"):  
    super().__init__()  
    self.size = size  
    self.max_size = max_size  
  
    self.interpolation = interpolation  
    # 平滑
    self.antialias = antialias

	def forward(self, img):  
		# size 只给一个 int 时，用短边匹配
		return F.resize(img, 
						self.size, 
						self.interpolation, 
						self.max_size, self.antialias)
```
## 3.4 RandomCrop
```python
class RandomCrop(torch.nn.Module):
	def __init__(self, size, padding=None, pad_if_needed=False, fill=0, padding_mode="constant"):  
	    super().__init__()  
	  
	    self.size = tuple(_setup_size(size, error_msg="Please provide only two dimensions (h, w) for size."))  
	  
	    self.padding = padding  
	    self.pad_if_needed = pad_if_needed  
	    self.fill = fill  
	    self.padding_mode = padding_mode

	def forward(self, img):  
	    if self.padding is not None:  
	        img = F.pad(img, self.padding, self.fill, self.padding_mode)  
	  
	    _, height, width = F.get_dimensions(img)  
	    # pad the width if needed  
	    if self.pad_if_needed and width < self.size[1]:  
			# 宽度不够用 0 填充
	        padding = [self.size[1] - width, 0]  
	        img = F.pad(img, padding, self.fill, self.padding_mode)  
	    # pad the height if needed  
	    if self.pad_if_needed and height < self.size[0]:  
	        padding = [0, self.size[0] - height]  
	        img = F.pad(img, padding, self.fill, self.padding_mode)  
	  
	    i, j, h, w = self.get_params(img, self.size)  
  
	    return F.crop(img, i, j, h, w)
```
## 3.5 RandomHorizontalFlip
```python
class RandomHorizontalFlip(torch.nn.Module):
	def __init__(self, p=0.5):  
	    super().__init__()  
	    # 翻转概率  
	    self.p = p

	def forward(self, img):  
	    if torch.rand(1) < self.p:  
	        return F.hflip(img)  
	    return img
```
## 3.6 ToTensor
```python
# 将 PIL 图像或 numpy.ndarray 转换为 tensor
class ToTensor:
	def __call__(self, pic):  
	    return F.to_tensor(pic)
```
## 3.7 ToPILImage
```python
class ToPILImage:
	def __init__(self, mode=None):  
		# color space & input depth
	    self.mode = mode
    def __call__(self, pic):
	    return F.to_pil_image(pic, self.mode)
```
# 4 torch.nn
## 4.1 Containers
### 4.1.1 Module
- 主要作用：
	- 记录模型需要的数据
	- 网络传播
	- 加载和保存模型数据
	- 设备转换，数据类型转换等
#### 4.1.1.1 属性
```python
# 所有 nn 模块的基类
class Module:
	# 表示 Module 处于 train 或 eval 模式，部分算子在不同模式下表现不同
	training: bool
	# 可学习的需要 GD 来更新的，如 weight, bias
	_parameters: Dict[str, Optional[Parameter]]
	# 不需要学习的参数
	_buffers: Dict[str, Optional[Tensor]]
	# 一些 hook 函数，用于在 BP FP 前后调用
	_forward_hooks: Dict[int, Callable]
	_backward_hooks: Dict[int, Callable]
	# 子 Module
	_modules: Dict[str, Optional['Module']]
```
#### 4.1.1.2 方法
```python
# 子类在继承时
# __init__ 方法中首先要调用父类的构造
# 子类需要重写 forward()
forward: Callable[..., Any] = _forward_unimplemented
def _forward_unimplemented(self, *input: Any) -> None:
	raise NotImplementedError(f"Module [{type(self).__name__}] is missing the required \"forward\" function")

# 将所有的参数和 buffer 移动到 CPU, GPU 上
# _apply() 会对所有子模块调用传入的 lambda 表达式
def cpu(self: T) -> T:
	return self._apply(lambda t: t.cpu())
	 
def cuda(self: T, device: Optional[Union[int, device]] = None) -> T:	
	return self._apply(lambda t: t.cuda(device))

# to() 可以原地修改 Module
def to(self, *args, **kwargs):
	# 解析参数
	device, dtype, non_blocking, convert_to_format = torch._C._nn._parse_to(*args, **kwargs)
	def convert(t):  
		if convert_to_format is not None and t.dim() in (4, 5):  
			return t.to(device, dtype if t.is_floating_point() or t.is_complex() else None, non_blocking, memory_format=convert_to_format)  
		return t.to(device, dtype if t.is_floating_point() or t.is_complex() else None, non_blocking)  
	# 对所有的子 Module 调用 convert
	return self._apply(convert)

# 转换数据类型
	type(dst_type)
	float()
	double()
	half() # 转为 half 类型
	bfloat16()

# 注册 hook 函数
#可以统计权重，对网络进行剪枝等，可以在不修改原模型 的情况下实现
register_xxx_hook(hook)

# 将自定义的参数保存在网络中
register_parameter()
# 保存在 buffer 中
register_buffer()

```
### 4.1.2 Container
```python
# Container 是 Module 的子类
class Container(Module):  
    def __init__(self, **kwargs: Any) -> None:  
        super().__init__()  
		# 调用父类的 add_module(): self._modules[name] = module
		# 将模块添加到子 Module 列表中
        for key, value in kwargs.items():  
            self.add_module(key, value)
```
### 4.1.3 Sequential
- 可以按顺序添加 Module
```python
class Sequential(Module):
	_modules: Dict[str, Module]  # type: ignore[assignment]

def __init__(self, *args):  
    super().__init__()  
    if len(args) == 1 and isinstance(args[0], OrderedDict):  
        for key, module in args[0].items():  
            self.add_module(key, module)  
    else:  
        for idx, module in enumerate(args):  
            self.add_module(str(idx), module)

# 也可以传入 OrderedDict
def __init__(self, arg: 'OrderedDict[str, Module]')

# 添加 Module
def append(self, module: Module) -> 'Sequential':    
	self.add_module(str(len(self)), module)  
    return self
```
### 4.1.4 其他类型的 Container
```python
# 以字典形式保存模块
class ModuleDict(Module):
	_modules: Dict[str, Module]  # type: ignore[assignment]  
	  
	def __init__(self, modules: Optional[Mapping[str, Module]] = None) -> None:  
		super().__init__()  
		if modules is not None:  
			self.update(modules)

# 以列表形式保存模块
class ModuleList(Module):
	_modules: Dict[str, Module]  # type: ignore[assignment]  
	
	def __init__(self, modules: Optional[Iterable[Module]] = None) -> None:  
	    super().__init__()  
	    if modules is not None:  
	        self += modules

# ParameterDict ParameterList 
```
# 5 Convolution Layer
- no padding, no stride(default 1)
<img src="D:\Doc\ob_note\images_dl\no_padding_no_strides.gif" style="zoom:60%;" />
- arbitrary padding, no stride
<img src="D:\Doc\ob_note\images_dl\arbitrary_padding_no_strides.gif" style="zoom:30%;" />
- half padding, no stride
<img src="D:\Doc\ob_note\images_dl\same_padding_no_strides.gif" style="zoom:40%;" />
- full padding, no stride
<img src="D:\Doc\ob_note\images_dl\full_padding_no_strides.gif" style="zoom:30%;" />
- 卷积的父类是 `_ConvNd`
```python
class _ConvNd(Module):
	in_channels: int
	out_channels: int
	kernel_size: Tuple[int, ...]  
	stride: Tuple[int, ...]  
	padding: Union[str, Tuple[int, ...]]
	padding_mode: str  
	weight: Tensor  
	bias: Optional[Tensor]
```
## 5.1 Conv2d
```python
class Conv2d(_ConvNd):
	def __init__(  
    self,  
    in_channels: int,  
    out_channels: int,  
    kernel_size: _size_2_t,  
    stride: _size_2_t = 1,  
    padding: Union[str, _size_2_t] = 0,  
    dilation: _size_2_t = 1,  
    groups: int = 1,  
    bias: bool = True,  
    padding_mode: str = 'zeros',  # TODO: refine this type  
    device=None,  
    dtype=None  
) -> None:
		kernel_size_ = _pair(kernel_size)  
		stride_ = _pair(stride)  
		padding_ = padding if isinstance(padding, str) else _pair(padding)

def _conv_forward(self, input: Tensor, weight: Tensor, bias: Optional[Tensor]):  
    if self.padding_mode != 'zeros':  
        return F.conv2d(F.pad(input, 
				        self._reversed_padding_repeated_twice, 
				        mode=self.padding_mode),  
                        weight, bias, self.stride,  
                        _pair(0), self.dilation, self.groups)  
    return F.conv2d(input, weight, bias, self.stride,
	    self.padding, self.dilation, self.groups)

def forward(self, input: Tensor) -> Tensor:  
    return self._conv_forward(input, self.weight, self.bias)
```
- 底层调用 `functional` 中的 `conv2d` 函数：
```python
torch.nn.functional.conv2d(input, weight, bias=None, stride=1, padding=0, dilation=1, groups=1)
```
- 参数：
	- **input**: `shape` 为 $(minibatch, in\_channels, iH, iW)$ 的 tensor
	- weight：`shape` 为 $(out\_channels, \frac{in_channels}{groups}, kH, kW)$ 的 `filters`
	- stride: 卷积核移动的步长
	- padding
## 5.2 创建一个卷积层
- `nn.Conv2d`
	- in_channels: 输入通道数
	- out_channels
	- kernel_size(int or tuple)
	- stride: 默认 1
	- padding: 默认 0
	- padding_mode: 默认 'zero'
	- dilation: 默认 1
	- bias：默认 True
```python
class MyModule(nn.Module):  
    def __init__(self):  
        super(MyModule, self).__init__()  
        self.conv1 = nn.Conv2d(in_channels=3,  
                               out_channels=6,   
                               kernel_size=3,   
                               stride=1, padding=0)  
    def forward(self, x):  
        return self.conv1(x)
```
# 6 Pooling Layer
- 最大池化的父类是 `_MaxPoolNd`
```python
class _MaxPoolNd(Module):  
    __constants__ = ['kernel_size', 'stride', 'padding', 'dilation',  
                     'return_indices', 'ceil_mode']  
    return_indices: bool  
    ceil_mode: bool  
  
    def __init__(self, kernel_size: _size_any_t, 
			    stride: Optional[_size_any_t] = None,  
                 padding: _size_any_t = 0, 
                 dilation: _size_any_t = 1,  
                 return_indices: bool = False, 
                 ceil_mode: bool = False) -> None:  
        super().__init__()  
        self.kernel_size = kernel_size  
        self.stride = stride if (stride is not None) else kernel_size  
        self.padding = padding  
        self.dilation = dilation  
        self.return_indices = return_indices  
        self.ceil_mode = ceil_mode
```
## 6.1 MaxPool2d
```python
class MaxPool2d(_MaxPoolNd):
	kernel_size: _size_2_t  
	stride: _size_2_t  
	padding: _size_2_t  
	dilation: _size_2_t  

	def forward(self, input: Tensor):  
	    return F.max_pool2d(input, self.kernel_size, 
						    self.stride,  
	                        self.padding, self.dilation, 
	                        ceil_mode=self.ceil_mode,  
	                        return_indices=self.return_indices)
```
- 创建一个 MaxPool2d 层：
	- kernel_size
	- stride
	- padding
	- ceil_mode: 默认 True，无 padding 遇到空值时正常计算
```python
class MyModule(nn.Module):  
    def __init__(self):  
        super(MyModule, self).__init__()  
        self.pool = nn.MaxPool2d(kernel_size=3,  
                                 ceil_mode=False)  
          
    def forward(self, input):  
        return self.pool(input)
```
## 6.2 AvgPool2d
- 平均池化的父类是 `_AvgPoolNd`
```python
class AvgPool2d(_AvgPoolNd):
	kernel_size: _size_2_t  
	stride: _size_2_t  
	padding: _size_2_t  
	ceil_mode: bool  
	count_include_pad: bool
	def __init__(self, kernel_size: _size_2_t, 
				stride: Optional[_size_2_t] = None, 
				padding: _size_2_t = 0,  
	             ceil_mode: bool = False) -> None:  
	    super().__init__()  
	    self.kernel_size = kernel_size  
	    self.stride = stride if (stride is not None) else kernel_size  
	    self.padding = padding  
	    self.ceil_mode = ceil_mode  
	    self.count_include_pad = count_include_pad  
	    self.divisor_override = divisor_override  
  
	def forward(self, input: Tensor) -> Tensor:  
	    return F.avg_pool2d(input, self.kernel_size, self.stride,  
	                        self.padding, self.ceil_mode, 
	                        self.count_include_pad, 
	                        self.divisor_override)
```
- 参数
	- kernel_size
	- stride
	- padding
	- ceil_mode
	- count_include_pad: 默认 True，在计算时包括 pad 中的 0
	- divisor_override：默认 None，用池化区域进行计算
# 7 Non-lineal Activation
## 7.1 ReLU
$$
ReLU(x) = max(0, x)
$$
```python
class ReLU(Module):
	inplace: bool  
  
	def __init__(self, inplace: bool = False):  
	    super().__init__()  
	    self.inplace = inplace  
	  
	def forward(self, input: Tensor) -> Tensor:  
	    return F.relu(input, inplace=self.inplace)
```
<img src="D:\Doc\ob_note\images_dl\activation\relu.png" style="zoom:60%;" />
## 7.2 Sigmoid
$$
Sigmoid(x)=\sigma(x)=\frac{1}{1+exp(-x)}
$$
```python
class Sigmoid(Module):
	def forward(self, input: Tensor) -> Tensor:  
    return torch.sigmoid(input)
```
<img src="D:\Doc\ob_note\images_dl\activation\sigmoid.png" style="zoom:60%;" />
# 8 其他层
## 8.1 Normalization Layer

![[norms.png]]
### 8.1.1 BatchNorm

<img src="D:\Doc\ob_note\images_dl\BN.png" style="zoom:50%;" />

- 针对某个 channel，对该 channel 的所有特征进行标准化
- 每个 `channel` 中像素值通常有不同的分布和范围，会导致不收敛，通过归一化将每个 `channel` 的分布标准化为 `Gaussian` 分布
- 理论上需要对每一层的 `feature map` 中的每个 `channel` 计算均值和方差，非常耗费时间，因此采用 `BN`，对当前的 `batch` 计算，并进行归一化
- 通常采用从 ImageNet 数据集中计算出的均值和方差
	- `transform.Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225])`
### 8.1.2 LayerNorm
- LN 对每个样本计算均值和方差，然后归一化
- LN 适用于 RNN，transformer 等，因为 sequence 可能是长度不一致的

<img src="D:\Doc\ob_note\images_dl\LN.png" style="zoom:50%;" />

```python
torch.nn.LayerNorm(normalized_shape, eps=1e-05, elementwise_affine=True, bias=True, device=None, dtype=None)
```
- InstanceNorm 与 LayerNorm
	- IN 通常作用在每个 channel 上，而 LN 作用在整个样本上
	- LN 经常用于 NLP 中，且会使用仿射变换

## 8.2 Linear Layer
## 8.3 Recurrent Layer
## 8.4 Transformer Layer
## 8.5 Dropout Layer
# 9 Loss 
## 9.1 L1Loss
- 计算 x 和 y 的 MAE
$$\ell(x,y)=L=\{l_1,...,l_N\}^T,~~~l_n=|x_n-y_n|$$
```python
class L1Loss(size_average=None,
			reduce=None,
			reduction='mean')
''' shape:
		input(*)
		target(*): same as input
		output: scalar
'''
```
## 9.2 MSELoss
- 计算 x 和 y 的均方 L2 范数
$$\ell(x,y)=L=\{l_1,...,l_N\}^T,~~~l_n=(x_n-y_n)^2$$
```python
class MSELoss(size_average=None,
			reduce=None,
			reduction='mean')
''' shape:
		input(*)
		target(*): same as input
		output: scalar
'''
```
## 9.3 CrossEntropyLoss
$$
loss\left( x,class \right) =-\log \left( \frac{\exp \left[ x\left[ class \right] \right]}{\sum_j{\exp \left( x\left[ j \right] \right)}} \right) =-x\left[ class \right] +\log \left( \sum_j{\exp \left( x\left[ j \right] \right)} \right) 
$$
```python
class CrossEntropyLoss(weight=None,
					size_average=None,
					ignore_index=-100,
					reduce=None,
					reduction='mean',
					label_smoothing=0.0)
''' shape:
		input: (C), (N, C) 
		target: (), (N)
		output: (), (N)
'''
```
# 10 Optimizer

![[optimizers.gif]]
- Adagrad, Adadelta 和 RMSprop 几乎直接朝正确方向前进
- Momentum 和 NAG 偏离了轨道，但 NAG 很快就可以纠正

- 优化器的基类是 Optimizer
```python
class Optimizer:
	# params: 需要优化的网络参数
	def __init__(self, 
				params: params_t, 
				defaults: Dict[str, Any]) -> None:  
	    self.defaults = defaults  
		# 一些 hook
	# 一次参数更新
	def step(self, closure: Callable[[], float]) -> float:
		...
```
## 10.1 SGD
```python
class SGD(Optimizer):
	def __init__(self, params, lr=required, momentum=0, 
				dampening=0, weight_decay=0, nesterov=False, *, 
				maximize: bool = False, 
				foreach: Optional[bool] = None,  
	            differentiable: bool = False):  
    defaults = dict(lr=lr, momentum=momentum, dampening=dampening,  
                    weight_decay=weight_decay, nesterov=nesterov,  
                    maximize=maximize, foreach=foreach,  
                    differentiable=differentiable)  
    if nesterov and (momentum <= 0 or dampening != 0):  
        raise ValueError("Nesterov momentum requires a momentum and zero dampening")  
    super().__init__(params, defaults)
```
- 参数
	- params(iterable)
	- lr(float)
	- momentum(float, optional)
		- SGD 在遇到沟壑时会陷入震荡
		- 在计算梯度时，当前时刻的梯度等于目前为止梯度的指数加权平均，即让梯度保留之前的一部分速度和方向，称为含动量的 SGD
		- Nesterov Accelerated Gradient 是对含动量 SGD 的一种改进，能够防止大幅震荡 
	- weight_decay(float, optional)
	- dampening
	- nesterov(bool, optional)
	- maxmize(bool, optional)