---
layout: post
title: AWS Ubuntu Install TensorFlow
disqus: y
share: y
categories: [Ai]
tags: [Aws, Tensorflow, Ubuntu]
---

Steps to install TensorFlow to AWS ubuntu node, some tips

AWS Part
---------------------
1. Go to AWS https://aws.amazon.com
2. Create the free Ubuntu node, I use "Ubuntu Server 16.04 LTS (HVM), SSD Volume Type"
3. Security setting, open 8888, 80, 22, ICMP ports

Python Part
---------------------
1. Must run source update in Ubuntu

```shell
$ sudo apt-get update
```
2. AWS Ubuntu defaul install python3 and python-minimal which is 2.7, we can manually install python2.7

```shell
$ sudo apt-get install python2.7
```
3. Install pip + Virtualenv(virtual machine include docker)

```shell
$ sudo apt-get install python-pip python-dev python-virtualenv
```
4. Install all the python Lib(very big)

```shell
$ sudo apt-get install python-numpy python-scipy python-matplotlib ipython ipython-notebook python-pandas python-sympy python-nose python-sklearn
```

TensorFlow Part
---------------------
1. Get TensorFlow package by virtualenv

```shell
$ virtualenv --system-site-packages ~/tensorflow
```
2. Active TensorFlow

```shell
$ source ~/tensorflow/bin/activate  # If using bash
```
3. Startup TensorFlow docker

```shell
$ sudo docker run -p 8888:8888 --name tensorflow-udacity -it gcr.io/tensorflow/udacity-assignments:0.6.0
```
if exist

```shell
$ sudo docker start -ai tensorflow-udacity
```

Visit Your TensorFlow
---------------------
1. visit http://{your AWS IP}:8888

Setup environment always annoying, Enjoy! 
