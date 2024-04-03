---
toc: true
title: "如何使用jwt内嵌Grafana的dashboard?"
categories:
  - devops
tags:
  - grfana
---

# 第一步: 生成jwk

https://mkjwk.org/

![](https://raw.githubusercontent.com/xiaoshenwei/xiaoshenwei.github.io/master/assets/images/image-2023-4-17_15-27-14.png)

保存中间 部分为 jwks.json

保存右侧部分为 pub.json 注意将其添加为{"keys": [{...}]} 此格式

# 第二步: 修改grafana.ini文件

```bash
#################################### Auth JWT ##########################
[auth.jwt]
enabled = true
header_name = X-JWT-Assertion
email_claim = email
username_claim = username
;jwk_set_url = https://foo.bar/.well-known/jwks.json
jwk_set_file = /path/to/pub.json
;cache_ttl = 60m
;expected_claims = {"aud": ["foo", "bar"]}
;key_file = /path/to/key/file
;role_attribute_path =
;role_attribute_strict = false
auto_sign_up = true
url_login = true
;allow_assign_grafana_admin = false
```

重启grafana

# 第三步: 使用代码基于jwks生成token

使用demo

```python
import json
import jwt
from jwt import PyJWKSet
 
with open("jwks.json", 'r') as f:
    jwk = PyJWKSet(json.load(f).get('keys'))
 
if __name__ == "__main__":
    print(jwk.keys[0].key, jwk.keys[0].key_id)
    payload = {
        "email": "xiaoshenwei@dxy.cn",
        "username": "xiaoshenwei",
        "role": "Viewer"
    }
    headers = {
        "kid": jwk.keys[0].key_id,
    }
    bt = jwt.encode(payload, key=jwk.keys[0].key, headers=headers, algorithm="RS256")
    print(bt)
```

# 第四步: 使用token 进行登录

https://grafana-test/?auth_token=xxxxxxxxxxxxxxxxx