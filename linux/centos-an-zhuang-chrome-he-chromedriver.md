---
description: centos安装chrome和chromedriver
---

# centos安装chrome和chromedriver

## chrome安装

```bash
wget https://dl.google.com/linux/direct/google-chrome-stable_current_x86_64.rpm # 下载 rpm
yum localinstall google-chrome-stable_current_x86_64.rpm # rpm安装
google-chrome-stable --version # 查看版本
```

## chromedriver安装

```bash
wget https://storage.googleapis.com/chrome-for-testing-public/124.0.6367.207/linux64/chromedriver-linux64.zip # 版本和chrome版本对应
unzip chromedriver-linux64.zip
```
