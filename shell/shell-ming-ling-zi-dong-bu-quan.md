# shell命令自动补全

## 背景

在使用k8s的时候，经常会用到kubelet命令，命令选项太多，不好记；因此考虑做一个自动补全

```bash
sudo yum -y install bash-completion
source /usr/share/bash-completion/bash_completion
sudo kubectl completion bash | sudo tee /etc/bash_completion.d/kubectl > /dev/null
sudo chmod a+r /etc/bash_completion.d/kubectl
```
