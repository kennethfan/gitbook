# kubectl 常用命令

1、获取对应服务的pods

```
kubectl get pods -o wide -n <命名空间> -l app=<服务名>
```

2、查看日志

```
kubectl logs -f <pod名称> -n <命名空间>
```
