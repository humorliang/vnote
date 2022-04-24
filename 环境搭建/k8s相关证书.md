# k8s 相关的证书

> 整个集群要使用统一的CA 证书，只需要在ansible控制端创建，然后分发给其他节点；为了保证安装的幂等性

## CA证书

**1. CA 配置文件**

示例：ca-config.json.j2 模板
```json
{
  "signing": {
    "default": {
      "expiry": "{{ CERT_EXPIRY }}"
    },
    "profiles": {
      "kubernetes": {
        "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ],
        "expiry": "{{ CERT_EXPIRY }}"
      }
    },
    "profiles": {
      "kcfg": {
        "usages": [
            "signing",
            "key encipherment",
            "client auth"
        ],
        "expiry": "{{ CUSTOM_EXPIRY }}"
      }
    }
  }
}
```
**说明：**
```
signing：表示该证书可用于签名其它证书；生成的 ca.pem 证书中 CA=TRUE；

server auth：表示可以用该 CA 对 server 提供的证书进行验证；

client auth：表示可以用该 CA 对 client 提供的证书进行验证；

profile kubernetes: 包含了server auth和client auth，所以可以签发三种不同类型证书；expiry 证书有效期，默认50年

profile kcfg: 在后面客户端kubeconfig证书管理中用到
```
**2. CA 证书签名请求**
示例： ca-csr.json.j2 模板
```json
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "HangZhou",
      "L": "XS",
      "O": "k8s",
      "OU": "System"
    }
  ],
  "ca": {
    "expiry": "876000h"
  }
}
```
**说明：**
`ca expiry` 指定ca证书的有效期，默认100年

**生成CA证书和私钥**
```sh
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

## kubeconfig 使用的admin 证书

**1. admin 证书签名请求 admin-csr.json**
```json
{
  "CN": "admin",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "HangZhou",
      "L": "XS",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
```

**生成admin 用户证书**
```sh
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin
```

## kube-proxy 使用的证书

**1. kube-proxy 证书请求**
示例： kube-proxy-csr.json
```json
{
  "CN": "system:kube-proxy",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "HangZhou",
      "L": "XS",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
```
**生成 system:kube-proxy 用户证书**
```sh
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-proxy-csr.json | cfssljson -bare kube-proxy
```


## etcd使用的证书

**1. etcd 证书请求**
示例: etcd-csr.json.j2 模板
```json
{
  "CN": "etcd",
  "hosts": [
{% for host in groups['etcd'] %}
    "{{ host }}",
{% endfor %}
    "127.0.0.1"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "HangZhou",
      "L": "XS",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
```
**说明：**
`etcd`使用对等证书，`hosts` 字段必须指定授权使用该证书的 `etcd` 节点 `IP`，这里枚举了所有`ectd`节点的地址

```sh
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes etcd-csr.json | cfssljson -bare etcd
```
   