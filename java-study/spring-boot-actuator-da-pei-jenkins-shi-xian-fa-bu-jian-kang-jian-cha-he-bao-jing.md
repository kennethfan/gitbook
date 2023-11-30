---
description: spring boot actuator搭配jenkins实现发布健康检查和报警
---

# spring boot actuator搭配jenkins实现发布健康检查和报警

## 背景 <a href="#springbootactuator-da-pei-jenkins-shi-xian-fa-bu-jian-kang-jian-cha-he-bao-jing-bei-jing" id="springbootactuator-da-pei-jenkins-shi-xian-fa-bu-jian-kang-jian-cha-he-bao-jing-bei-jing"></a>

当前项目使用jenkins发布，服务重启使用systemctl restart xxx.service，但是具体服务有没有起来，jenkins并没有感知，需要人工介入观察

## 目标 <a href="#springbootactuator-da-pei-jenkins-shi-xian-fa-bu-jian-kang-jian-cha-he-bao-jing-mu-biao" id="springbootactuator-da-pei-jenkins-shi-xian-fa-bu-jian-kang-jian-cha-he-bao-jing-mu-biao"></a>

实现jenkins自动检查服务启动状态，如果失败并报警

## 方案 <a href="#springbootactuator-da-pei-jenkins-shi-xian-fa-bu-jian-kang-jian-cha-he-bao-jing-fang-an" id="springbootactuator-da-pei-jenkins-shi-xian-fa-bu-jian-kang-jian-cha-he-bao-jing-fang-an"></a>

1、使用spring boot actuator实现服务状态观察，提供http接口方式

2、curl请求健康检查接口，解析返回结果感知服务状态

3、如果失败，通过钉钉报警

## 实施 <a href="#springbootactuator-da-pei-jenkins-shi-xian-fa-bu-jian-kang-jian-cha-he-bao-jing-shi-shi" id="springbootactuator-da-pei-jenkins-shi-xian-fa-bu-jian-kang-jian-cha-he-bao-jing-shi-shi"></a>

### 服务集成spring boot actuator <a href="#springbootactuator-da-pei-jenkins-shi-xian-fa-bu-jian-kang-jian-cha-he-bao-jing-fu-wu-ji-cheng-sprin" id="springbootactuator-da-pei-jenkins-shi-xian-fa-bu-jian-kang-jian-cha-he-bao-jing-fu-wu-ji-cheng-sprin"></a>

maven坐标

```xml
<dependency>    
	<groupId>org.springframework.boot</groupId>    
	<artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

如果需要显示components详情，需要在配置文件加上配置

```yaml
management:
  endpoint:
    health:
      show-details: always
```

健康检查地址：http://ip:port/${server.servlet.context-path}/actuator/health

比如：manager-api服务，ip:127.0.0.1，端口：8086，context-path：/manager，那么健康检查地址就是[http://127.0.0.1:8086/manager/actuator/health](http://127.0.0.1:8086/manager/actuator/health)

可以curl访问下，响应如下

```json
{"status":"UP","components":{"db":{"status":"UP","details":{"database":"MySQL","validationQuery":"isValid()"}},"discoveryComposite":{"status":"UP","components":{"discoveryClient":{"status":"UP","details":{"services":["api-gateway","infrastructure-api","manager-api","ticket-api","user-api"]}}}},"diskSpace":{"status":"UP","details":{"total":62799216640,"free":51376033792,"threshold":10485760,"exists":true}},"nacosConfig":{"status":"UP"},"nacosDiscovery":{"status":"UP"},"ping":{"status":"UP"},"reactiveDiscoveryClients":{"status":"UP","components":{"Simple Reactive Discovery Client":{"status":"UP","details":{"services":[]}}}},"redis":{"status":"UP","details":{"version":"6.2.6"}},"refreshScope":{"status":"UP"},"sentinel":{"status":"UP","details":{"dataSource":{},"enabled":true,"dashboard":{"description":"dashboard isn't configured","status":"UNKNOWN"}}}}}
```

如果没有开启show-details

响应比较简单

```json
{"status":"UP"}
```

status = UP表示服务启动成功

### jenkins检查脚本 <a href="#springbootactuator-da-pei-jenkins-shi-xian-fa-bu-jian-kang-jian-cha-he-bao-jing-jenkins-jian-cha-jia" id="springbootactuator-da-pei-jenkins-shi-xian-fa-bu-jian-kang-jian-cha-he-bao-jing-jenkins-jian-cha-jia"></a>

```groovy
print("${server_name} heatlh-check begin") // debug
def url = "http://127.0.0.1:" + service_map[server_name]["port"] + service_map[server_name]["path"] + "/actuator/health" // 构建健康检查地址；因为防火墙的原因，并不是直接从jenkins直接curl，而是通过ssh远程执行，具体见下文
def i = 0
def success = false
while (i < 60) { // 服务启动需要一定时间，因此需要重试等待
    i += 1
    print("${server_name} heatlh-check try $i")
    def response = sh script: """ssh -p ${__Deploy_SSH_Port} ${__Deploy_Account}@${__Deploy_IP} "curl -s -w'%{http_code}' '$url' || ls > /dev/null " """, returnStdout: true, returnStatus: false // returnStdout: true 表示返回标准输出，这样才可以使用变量接收；服务未起来时curl命令返回状态码7会导致脚本退出，因此加上 || xxx保证返回状态码为0
    print(response)
    if (response == "000") { // curl失败时，http_status = 000
        print("${server_name} heatlh-check response null")
        sleep(5)
        continue
    }
 
    def status_code = response.substring(response.size() - 3, response.size()) // 解析http_status
    if (status_code == "000") {
        print("${server_name} heatlh-check response null")
        sleep(5)
        continue
    }
 
    success = true
 
    if (status_code == "404") {
        print("${server_name} 未配置heatlh-check")
        break
    }
 
    if (status_code != "200" && status_code != "503") {
        print("${server_name} heatlh-check错误")
        break
    }
 
    def body = response.substring(0, response.size() - 3) // 解析body
 
    def json = readJSON text: body // json解析
    if ("UP" == json.status) {
        print("${server_name} heatlh-check passed")
        break
    }
 
    // 启动失败
    def message = "[监控报警]\n服务部署失败\njob=${JOB_NAME}\nserver_name=${server_name}\nip=${__Deploy_IP}\ndetail=${url}"
    def param = ["msgtype": "text", "text": ["content": message]]
    def data = writeJSON returnText: true, json: param // object转json
    sh """curl -H 'Content-Type: application/json; charset=utf-8' -d '${data}' ${ding_url} """  // 通过钉钉webhook发送报警
    break
}
 
if (!success) {
    // 启动失败
    def message = "[监控报警]\n服务部署失败\njob=${JOB_NAME}\nserver_name=${server_name}\nip=${__Deploy_IP}\ndetail=${url}"
    def param = ["msgtype": "text", "text": ["content": message]]
    def data = writeJSON returnText: true, json: param
    sh """curl -H 'Content-Type: application/json; charset=utf-8' -d '${data}' ${ding_url} """
}
```
