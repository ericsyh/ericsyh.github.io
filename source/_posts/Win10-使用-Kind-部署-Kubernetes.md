---
title: Win10 使用 Kind 部署 Kubernetes
date: 2020-11-01 18:16:28
tags:
	- Kubernetes
	- Kind
---

## 背景
Win10 的 WSL2 支撑了在 Linux 子系统中使用 Docker，尝试在 WSL2 环境下通过 [Kind](https://github.com/kubernetes-sigs/kind) 来部署本地 Kubernetes 环境。

## 操作步骤

### WSL2 安装
详细文档参考 [Installation Instructions for WSL2](https://docs.microsoft.com/en-us/windows/wsl/install-win10)
1. 在Windows程序中，选择启用 "适用于 Linux 的 Windows 子系统"
2. 管理员身份打开 PowerShell 并运行 `dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart`
3. 管理员身份打开 PowerShell 并运行 `wsl --set-default-version 2`
4. 打开 Microsoft Store，并选择自己偏好的 Linux 分发版

### Docker-desktop 安装
文档参考 [docker-for-windows](https://docs.docker.com/docker-for-windows/wsl/) 进行 Win10 Docker-desktop 安装

### Kind 安装
1. 进入 Win10 的 Linux 子系统，在子系统内安装 Kind binary
```
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.9.0/kind-linux-amd64
chmod +x ./kind
mv ./kind /some-dir-in-your-PATH/kind
```
2. 准备 Kind 所需的 kindest/node 镜像 `docker pull kindest/node:v1.19.1`
3. 使用 Kind 命令创建本地 Kubernetes `kind create cluster`
```
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.19.1) 🖼
 ✓ Preparing nodes 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind

Thanks for using kind! 😊
```
4. 使用 kubectl 部署 deployment `kubectl create deployment nginx --image=nginx`
```
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   1/1     1            0           63s
```