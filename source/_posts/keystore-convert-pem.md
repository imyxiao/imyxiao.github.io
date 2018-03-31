---
title: 转换java keytools的keystore证书到OPENSSL的PEM格式文件
author: 肖哥
avatar: /yxiao.jpg
authorLink: https://blog.imyxiao.com
authorAbout: https://blog.imyxiao.com/about
authorDesc: Java Developer
date: 2018-02-08 15:20:48
categories: java
tags:
  - keystore
  - pem
  - nginx
  - certificate
  - https
description: 使用https直接连接后端的tomcat,证书是通过JDK自带的keytool生成的,现决定迁移到nginx做ssl，但是keytool生成的证书都是二进制data，nginx使用的是OPENSSL标准的PEM+key文件，即ascii文本格式的密钥，所以需要作证书转换
---
使用https直接连接后端的tomcat,证书是通过JDK自带的keytool生成的.假如keytool生成密钥对`server.keystore`如下:

```bash
 keytool -genkey -alias tomcat-server -keyalg RSA -keypass xxxxxxx -storepass xxxxxxx -keystore server.keystore
```

现决定迁移到nginx做ssl，但是keytool生成的证书都是二进制data，nginx使用的是OPENSSL标准的PEM+key文件，即ascii文本格式的密钥，所以需要作证书转换．

首先将服务器证书导出为证书文件`server.cer`:

```bash
keytool -export -alias tomcat-server -storepass xxxxxxx -file server.cer -keystore server.keystore
```

cer文件到PEM文件的转换较简单。这两者都是X509证书，编码不同，使用openssl工具即可：

```bash
openssl x509 -inform der -in server.cer -out server.pem
```
<!-- more --> 
至于keystore转换就比较麻烦，首先使用`http://download.csdn.net/detail/cwxzz/1072684`这里的工具，PFX格式证书和JAVA keyStore证书相互转换，先将keystore转换为PFX证书。修改java代码，填入keystore路径，生成文件的路径，KEYSTORE_PASSWORD。运行后得到PFX证书`server.pfx`

```java
public static final String PFX_KEYSTORE_FILE = "server.pfx";
    public static final String KEYSTORE_PASSWORD = "xxxxxxx";
    public static final String JKS_KEYSTORE_FILE = "server.keystore";

    private static void coverToPfx() {
        try {
            KeyStore inputKeyStore = KeyStore.getInstance("JKS");
            FileInputStream fis = new FileInputStream(JKS_KEYSTORE_FILE);
            char[] nPassword;

            if (KEYSTORE_PASSWORD.trim().equals("")) {
                nPassword = null;
            } else {
                nPassword = KEYSTORE_PASSWORD.toCharArray();
            }

            inputKeyStore.load(fis, nPassword);
            fis.close();

            KeyStore outputKeyStore = KeyStore.getInstance("PKCS12");
            outputKeyStore.load(null, KEYSTORE_PASSWORD.toCharArray());
            Enumeration enums = inputKeyStore.aliases();
            while (enums.hasMoreElements()) {
                String keyAlias = (String) enums.nextElement();
                System.out.println("alias=[" + keyAlias + "]");
                if (inputKeyStore.isKeyEntry(keyAlias)) {
                    Key key = inputKeyStore.getKey(keyAlias, nPassword);
                    Certificate[] certChain = inputKeyStore.getCertificateChain(keyAlias);
                    outputKeyStore.setKeyEntry(keyAlias, key, KEYSTORE_PASSWORD.toCharArray(), certChain);
                }
            }
            FileOutputStream out = new FileOutputStream(PFX_KEYSTORE_FILE);
            outputKeyStore.store(out, nPassword);
            out.close();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
```

接下来使用openssl从PFX中提取私钥

```bash
openssl pkcs12 -in server.pfx -nocerts -nodes -out server.key
```

这里也需要输入生成证书时使用的密码。这样ascii格式的key文件`server.key`也可以使用了。

如果私钥带有密码,则需要移除密码:

```bash
openssl rsa -in server.key -out server.origin.key
```

nginx配置ssl示例(ssl开头的属性与证书配置有直接关系):

```bash
server {
    listen 443;
    server_name localhost;
    ssl on;
    root html;
    index index.html index.htm;
    ssl_certificate   server.pem;
    ssl_certificate_key  server.key;
    ssl_session_timeout 5m;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_prefer_server_ciphers on;
    location / {
        root html;
        index index.html index.htm;
    }
}
```

公钥提取可用如下方法:

```java
keytool -list -rfc --keystore server.keystore | openssl x509 -inform pem -pubkey
```

输入密码,即可输出公钥及证书

引用:

> - [转换java keytools的keystore证书到OPENSSL的PEM格式文件](https://www.cnblogs.com/interdrp/p/4880891.html)
> - [PFX格式证书和JAVA keyStore证书相互转换](http://download.csdn.net/download/cwxzz/1072684)
