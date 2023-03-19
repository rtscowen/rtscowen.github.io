---
layout: post
title: (Coming Soon) TorchInductor Deep Dive
author: rob
---
[IN PROGRESS] Deep dive into TorchInductor, the deep learning compiler underpinning that generates fast code for multiple accelerators and backends.

I think there will be a general trend towards using intermediate representations as the basis for speeding up NN training and inference. In this post, I'll look under the hood of TorchInductor, one of the technologies underpinning `torch.compile` in PyTorch 2.0