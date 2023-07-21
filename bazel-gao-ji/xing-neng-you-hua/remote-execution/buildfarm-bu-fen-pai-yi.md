# Buildfarm部分排疑

Buildfarm的使用实际上是有部分问题的，如果想把Buildfarm投入生产，建议注意下面几点

### grpc的负载均衡

和nginx的负载均衡不一样，grpc破坏了标准的连接级负载均衡，Kubernetes 提供的负载均衡本身是基于连接级别的负载均衡。这是因为 gRPC 构建于 HTTP/2 之上，而 HTTP/2 被设计为具有单个长期 TCP 连接，所有请求都在该连接上进行多路复用，这意味着多个请求可以在任何时间点在同一连接上处于活动_状态_。因此如果希望对buildfarm的服务做负载均衡，最好启用kubernetes的ingress机制

### Buildfarm的Liveness检查

Buildfarm从1.15到2.3.1目前已经很稳定了，但是在机器压力过大或者inode不够用的情况下还是会出现问题，这些问题的统一表现是buildfarm的worker，java创建大量线程，因此可以根据该特征写一个livenessprobe检查

````yaml
```yaml
        livenessProbe:
          exec:
            command:
            - /bin/bash
            - -c
            - |-
              java_thread_count=$(ps huH p $(ps -C java -o pid=) | grep java | wc -l);
              if [ "$java_thread_count" -gt "1000" ]; then
                exit 1
              fi
          failureThreshold: 12
          initialDelaySeconds: 300
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 3
```
````
