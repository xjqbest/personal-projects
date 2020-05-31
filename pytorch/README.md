
## DataLoader

[https://pytorch.org/docs/master/data.html](https://pytorch.org/docs/master/data.html)

[dataloader代码](https://github.com/pytorch/pytorch/blob/0b868b19063645afed59d6d49aff1e43d1665b88/torch/utils/data/dataloader.py)

DataSet: 组织数据集。map-style 和 iterable-style。

sampler: 承接DataSet和多进程reader。可以自定义一个batch中数据的顺序。

<img src="./images/dataloader.png" width="80%" height="80%">

