# torch-summary

Keras has a neat API to view the visualization of the model which is very helpful while debugging your network. In this project, we attempt to do the same in PyTorch. The goal is to provide information complementary to what is provided by `print(your_model)` in PyTorch.

This is a rewritten version of the original torchsummary and torchsummaryX projects by @sksq96 and @nmhkahn.
There are quite a few pull requests on the original project (which hasn't been updated in over a year), so I decided to take a stab at improving and consolidating some of the features.

This version now supports:
- RNNs, LSTMs, and other recursive layers
- Branching output to explore model layers using specified depths
- Returns ModelStatistics object to access summary data
- Configurable columns of returned data

Other features include:
- Verbose mode to show specific weights and bias layers
- Accepts either input data or simply the input shape to work!
- Customizable widths and batch dimension.
- More comprehensive testing using pytest


# Usage
`pip install torch-summary`

or

`git clone https://github.com/tyleryep/torch-summary.git`


```python
from torchsummary import summary
summary(your_model, input_data=(C, H, W))
```

# Documentation
```python
"""
Summarize the given PyTorch model. Summarized information includes:
    1) output shape,
    2) kernel shape,
    3) number of the parameters
    4) operations (Mult-Adds)
Args:
    model (Module): Model to summarize
    input_data (Sequence of Sizes or Tensors):
        Example input tensor of the model (dtypes inferred from model input).
        - OR -
        Shape of input data as a List/Tuple/torch.Size (dtypes must match model input,
        default to FloatTensors). NOTE: For scalars, use torch.Size([]).
    use_branching (bool): Whether to use the branching layout for the printed output.
    max_depth (int): number of nested layers to traverse (e.g. Sequentials)
    verbose (int):
        0 (quiet): No output
        1 (default): Print model summary
        2 (verbose): Show weight and bias layers in full detail
    col_names (List): specify which columns to show in the output. Currently supported:
        ['output_size', 'num_params', 'kernel_size', 'mult_adds']
    col_width (int): width of each column
    dtypes (List or None): for multiple inputs or args, must specify the size of both inputs.
        You must also specify the types of each parameter here.
    batch_dim (int): batch_dimension of input data
    args, kwargs: Other arguments used in `model.forward` function
Return:
    ModelStatistics object
        (see model_statistics.py for details on how to access the summary data)

"""
```


# Examples
## Get Model Summary as String
```python
model_stats = summary(your_model, input_data=(C, H, W), verbose=0)
summary_str = str(model_stats)
```

## CNN for MNIST

```python
import torch
import torch.nn as nn
import torch.nn.functional as F
from torchsummary import summary

class CNN(nn.Module):
    def __init__(self):
        super().__init__()
        self.conv1 = nn.Conv2d(1, 10, kernel_size=5)
        self.conv2 = nn.Conv2d(10, 20, kernel_size=5)
        self.conv2_drop = nn.Dropout2d(0.3)
        self.fc1 = nn.Linear(320, 50)
        self.fc2 = nn.Linear(50, 10)

    def forward(self, x):
        x = F.relu(F.max_pool2d(self.conv1(x), 2))
        x = F.relu(F.max_pool2d(self.conv2_drop(self.conv2(x)), 2))
        x = x.view(-1, 320)
        x = F.relu(self.fc1(x))
        x = self.fc2(x)
        return F.log_softmax(x, dim=1)


model = CNN()
summary(model, (1, 28, 28))
```


```
------------------------------------------------------------------------------------------
Layer (type:depth-idx)                   Output Shape              Param #
==========================================================================================
├─Conv2d: 1-1                            [-1, 10, 24, 24]          260
├─Conv2d: 1-2                            [-1, 20, 8, 8]            5,020
├─Dropout2d: 1-3                         [-1, 20, 8, 8]            --
├─Linear: 1-4                            [-1, 50]                  16,050
├─Linear: 1-5                            [-1, 10]                  510
==========================================================================================
Total params: 21,840
Trainable params: 21,840
Non-trainable params: 0
------------------------------------------------------------------------------------------
Input size (MB): 0.00
Forward/backward pass size (MB): 0.05
Params size (MB): 0.08
Estimated Total Size (MB): 0.14
------------------------------------------------------------------------------------------
```


## ResNet

```python
import torchvision
from torchsummary import summary


model = torchvision.models.resnet50()
summary(model, (3, 224, 224))
```


```
------------------------------------------------------------------------------------------
Layer (type:depth-idx)                   Output Shape              Param #
==========================================================================================
├─Conv2d: 1-1                            [-1, 64, 112, 112]        9,408
├─BatchNorm2d: 1-2                       [-1, 64, 112, 112]        128
├─ReLU: 1-3                              [-1, 64, 112, 112]        --
├─MaxPool2d: 1-4                         [-1, 64, 56, 56]          --
├─Sequential: 1-5                        [-1, 256, 56, 56]         --
|    └─Bottleneck: 2-1                   [-1, 256, 56, 56]         --
|    |    └─Conv2d: 3-1                  [-1, 64, 56, 56]          4,096
|    |    └─BatchNorm2d: 3-2             [-1, 64, 56, 56]          128
|    |    └─ReLU: 3-3                    [-1, 64, 56, 56]          --
|    |    └─Conv2d: 3-4                  [-1, 64, 56, 56]          36,864
|    |    └─BatchNorm2d: 3-5             [-1, 64, 56, 56]          128
|    |    └─ReLU: 3-6                    [-1, 64, 56, 56]          --
|    |    └─Conv2d: 3-7                  [-1, 256, 56, 56]         16,384
|    |    └─BatchNorm2d: 3-8             [-1, 256, 56, 56]         512
|    |    └─Sequential: 3-9              [-1, 256, 56, 56]         --
|    |    └─ReLU: 3-10                   [-1, 256, 56, 56]         --

  ...
  ...
  ...

├─AdaptiveAvgPool2d: 1-9                 [-1, 2048, 1, 1]          --
├─Linear: 1-10                           [-1, 1000]                2,049,000
==========================================================================================
Total params: 60,192,808
Trainable params: 60,192,808
Non-trainable params: 0
------------------------------------------------------------------------------------------
Input size (MB): 0.57
Forward/backward pass size (MB): 344.16
Params size (MB): 229.62
Estimated Total Size (MB): 574.35
------------------------------------------------------------------------------------------


```


## Multiple Inputs

```python
import torch
import torch.nn as nn
from torchsummary import summary


class SimpleConv(nn.Module):
    def __init__(self):
        super().__init__()
        self.features = nn.Sequential(
            nn.Conv2d(1, 1, kernel_size=3, stride=1, padding=1),
            nn.ReLU(),
        )

    def forward(self, x, y):
        x1 = self.features(x)
        x2 = self.features(y)
        return x1, x2


model = SimpleConv()
summary(model, [(1, 16, 16), (1, 28, 28)])
```


```
----------------------------------------------------------------
        Layer (type)               Output Shape         Param #
================================================================
            Conv2d-1            [-1, 1, 16, 16]              10
              ReLU-2            [-1, 1, 16, 16]               0
            Conv2d-3            [-1, 1, 28, 28]              10
              ReLU-4            [-1, 1, 28, 28]               0
================================================================
Total params: 20
Trainable params: 20
Non-trainable params: 0
----------------------------------------------------------------
Input size (MB): 0.77
Forward/backward pass size (MB): 0.02
Params size (MB): 0.00
Estimated Total Size (MB): 0.78
----------------------------------------------------------------
```

# References
- Thanks to @sksq96, @nmhkahn, and @sangyx for providing the original code this project was based off of.
- For Model Size Estimation @jacobkimmel ([details here](https://github.com/sksq96/pytorch-summary/pull/21))
