---
layout: post
title: "使用Python和Iris数据集搭建最简单的AI分类器"
date:   2022-10-10
tags: [AI,Python]
comments: true
author: nullptr-0
---

第一篇博客呀 :-)

## 环境

- Visual Studio 2022

- Python 3.9 64bit

## 代码

```
import numpy as np
import torch
from sklearn import datasets
import torch.nn as nn
from sklearn.model_selection import train_test_split
import matplotlib.pyplot as plt
#获得神奇的iris数据集
dataset = datasets.load_iris()
#按照二八比例划分测试集和数据集数据
input, x_test, label, y_test = train_test_split(dataset.data,dataset.target,test_size=0.2)
#利用pytorch把数据张量化,
input = torch.FloatTensor(input)
label = torch.LongTensor(label)
x_test = torch.FloatTensor(x_test)
y_test = torch.LongTensor(y_test)
label_size = int(np.array(label.size()))
#搭建神经网络 它有着两个隐藏层,一个输出层
#两个隐藏层均使用线性模型和relu激活函数 输出层使用softmax函数(dim参数设为1)
class NET(nn.Module):
    def __init__(self, n_feature, n_hidden1, n_hidden2, n_output):
        super(NET, self).__init__()
        self.hidden1 = nn.Linear(n_feature,n_hidden1)
        self.relu1 = nn.ReLU()
        self.hidden2 =nn.Linear(n_hidden1,n_hidden2)
        self.relu2 = nn.ReLU()
        self.out = nn.Linear(n_hidden2,n_output)
        self.softmax =nn.Softmax(dim=1)
#前向传播函数
    def forward(self, x):
        hidden1 = self.hidden1(x)
        relu1 = self.relu1(hidden1)
        hidden2 = self.hidden2(relu1)
        relu2 = self.relu2(hidden2)
        out = self.out(relu2)
        return out
#测试函数
    def test(self, x, y):
        y_pred = self.forward(x)
        y_predict = self.softmax(y_pred)
        prediction = torch.max(y_predict, 1)[1]
        pred_y = prediction.numpy()
        target_y = y.numpy()
        accuracy = float((pred_y == target_y).astype(int).sum()) / float(target_y.size)
        return accuracy
#定义网络结构以及损失函数
net = NET(n_feature=4, n_hidden1=50, n_hidden2=100, n_output=3)
#选择Adam优化器
optimizer = torch.optim.Adam(net.parameters(),lr = 0.001)
#交叉熵损失函数
loss_func = torch.nn.CrossEntropyLoss()
costs = []
accuracys = []
#训练次数
Epoch=10000
# 训练网络
for epoch in range(Epoch):
    cost = 0
#利用forward和损失函数获得out(输出)和loss(损失)
    out = net(input)
    loss = loss_func(out,label)
#clear gradients for next train
    optimizer.zero_grad()
#反向传播 并更新所有参数
    loss.backward()
    optimizer.step()
    cost = cost + loss.cpu().detach().numpy()
    costs.append(cost / label_size)
    accuracys.append(net.test(x_test, y_test)*100)
#可视化
#plt.plot(costs)
plt.plot(accuracys)
plt.show()
```
