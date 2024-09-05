# 问题

多租户集群中，如果A租户将镜像 拉取到node-01节点上， 租户B知道了 这个镜像名称，设置拉取策略为**IfNotPresent** 将直接启动这个阶段，如何解决？

# 准入控制器

[准入控制器](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/admission-controllers/) 是一段代码，它会在请求通过认证和鉴权之后、对象被持久化之前拦截到达 API 服务器的请求。

准入控制器可以执行**验证（Validating）** 和/或**变更（Mutating）** 操作。 变更（mutating）控制器可以根据被其接受的请求更改相关对象；验证（validating）控制器则不行。

![](https://kubernetes.io/images/blog/2019-03-21-a-guide-to-kubernetes-admission-controllers/admission-controller-phases.png)

准入控制过程分为两个阶段。

- 第一阶段，运行变更准入控制器。
- 第二阶段，运行验证准入控制器。

由于准入控制器是内置在 `kube-apiserver` 中的，这种情况下就限制了admission controller的可扩展性。在这种背景下，kubernetes提供了一种可扩展的准入控制器 `extensible admission controllers`，这种行为叫做动态准入控制 `Dynamic Admission Control`，而提供这个功能的就是 `admission webhook` 。

# 动态准入控制

在 Kubernetes apiserver 中包含两个特殊的准入控制器：`MutatingAdmissionWebhook`和`ValidatingAdmissionWebhook`。这两个控制器将发送准入请求到外部的 HTTP 回调服务并接收一个准入响应。如果启用了这两个准入控制器，Kubernetes 管理员可以在集群中创建和配置一个 admission webhook

## 查看下请求是什么样子的

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: "pod-debug.example.com"
webhooks:
- name: "pod-debug.example.com"
  rules:
  - apiGroups: [""]
    apiVersions: ["v1"]
    operations: ["CREATE"]
    resources: ["pods"]
    scope: "Namespaced"
  clientConfig:
    url: "https://192.168.198.183:443/debug"
    caBundle: "base64 /tmp/ssl/cert.pem"
  admissionReviewVersions: ["v1"]
  sideEffects: None
  timeoutSeconds: 5
```

```go
package main

import (
	"crypto/tls"
	"fmt"
	"github.com/gin-gonic/gin"
	"net/http"
)

const (
	tlsCertFile = "/tmp/ssl/cert.pem"
	tlsKeyFile  = "/tmp/ssl/key.pem"
)

func handleDebugRequest(c *gin.Context) {
	fmt.Println(c.Request.URL.Path)
	fmt.Println(c.Request.Header)
	// 打印请求体
	body := make([]byte, c.Request.ContentLength)
	c.Request.Body.Read(body)
	fmt.Println("Request Body:", string(body))

	c.JSON(200, gin.H{
		"message": "debug",
	})
}

func main() {
	r := gin.Default()

	r.GET("/ping", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "pong",
		})
	})
	r.POST("/debug", handleDebugRequest)
	sCert, err := tls.LoadX509KeyPair(tlsCertFile, tlsKeyFile)
	if err != nil {
		panic(err)
	}
	tlsConfig := &tls.Config{
		Certificates: []tls.Certificate{sCert},
		// TODO: uses mutual tls after we agree on what cert the apiserver should use.
		// ClientAuth:   tls.RequireAndVerifyClientCert,
	}

	s := &http.Server{
		Addr:      ":443",
		Handler:   r,
		TLSConfig: tlsConfig,
	}
	err = s.ListenAndServeTLS("", "")
	if err != nil {
		panic(err)
	}
}

```

Webhook 处理由 API 服务器发送的 `AdmissionReview` 请求，并且将其决定作为 `AdmissionReview` 对象以相同版本发送回去。

[请求](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/extensible-admission-controllers/#request)

[响应](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/extensible-admission-controllers/#response)



## deployment 副本数限制

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: "limit-deployment-replicas.example.com"
webhooks:
- name: "limit-deployment-replicas.example.com"
  rules:
  - apiGroups: ["apps"] 
    apiVersions: ["v1"]
    operations: ["CREATE"]
    resources: ["deployments"]
    scope: "Namespaced"
  clientConfig:
    url: "https://192.168.198.183:443/limit-deployment-replicas"
    caBundle: "base64 /tmp/ssl/cert.pem"
  admissionReviewVersions: ["v1"]
  sideEffects: None
  timeoutSeconds: 5
```

```go
package main

import (
	"crypto/tls"
	"encoding/json"
	"fmt"
	"github.com/gin-gonic/gin"
	"io"
	v1admission "k8s.io/api/admission/v1"
	appv1 "k8s.io/api/apps/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/apimachinery/pkg/runtime/serializer"
	"k8s.io/klog/v2"
	"net/http"
)

const (
	tlsCertFile = "/tmp/ssl/cert.pem"
	tlsKeyFile  = "/tmp/ssl/key.pem"
)

func handleDebugRequest(c *gin.Context) {
	fmt.Println(c.Request.URL.Path)
	fmt.Println(c.Request.Header)
	// 打印请求体
	body := make([]byte, c.Request.ContentLength)
	c.Request.Body.Read(body)
	fmt.Println("Request Body:", string(body))

	c.JSON(200, gin.H{
		"message": "debug",
	})
}

func handleLimitDeploymentReplicas(c *gin.Context) {
	var body []byte
	if c.Request.Body != nil {
		if data, err := io.ReadAll(c.Request.Body); err == nil {
			body = data
		}
	}

	var admission v1admission.AdmissionReview
	codefc := serializer.NewCodecFactory(runtime.NewScheme())
	decoder := codefc.UniversalDeserializer()
	_, _, err := decoder.Decode(body, nil, &admission)

	if err != nil {
		msg := fmt.Sprintf("Request could not be decoded: %v", err)
		klog.Error(msg)
		c.JSON(http.StatusBadRequest, gin.H{
			"message": msg,
		})
		return
	}
	if admission.Request == nil {
		klog.Error(fmt.Sprintf("admission review can't be used: Request field is nil"))
		c.JSON(http.StatusBadRequest, gin.H{
			"message": "admission review can't be used: Request field is nil",
		})
		return
	}
	// 获取请求体
	req := admission.Request
	var admissionResp v1admission.AdmissionReview
	admissionResp.APIVersion = admission.APIVersion
	admissionResp.Kind = admission.Kind
	klog.Infof("AdmissionReview for Kind=%v, Namespace=%v Name=%v UID=%v Operation=%v",
		req.Kind.Kind, req.Namespace, req.Name, req.UID, req.Operation)
	// 解析请求体, 转换为deployment对象
	var deployment appv1.Deployment
	if err = json.Unmarshal(req.Object.Raw, &deployment); err != nil {
		respStructure := v1admission.AdmissionResponse{Result: &metav1.Status{
			Message: fmt.Sprintf("could not unmarshal resouces review request: %v", err),
			Code:    http.StatusInternalServerError,
		}}
		klog.Error(fmt.Sprintf("could not unmarshal resouces review request: %v", err))
		if _, err = json.Marshal(respStructure); err != nil {
			klog.Error(fmt.Errorf("could not unmarshal resouces review response: %v", err))
			c.JSON(http.StatusInternalServerError, gin.H{
				"message": fmt.Errorf("could not unmarshal resouces review response: %v", err).Error(),
			})
			return
		}
		c.JSON(http.StatusBadRequest, gin.H{
			"message": fmt.Errorf("could not unmarshal resouces review response: %v", err).Error(),
		})
		return
	}
	respStructure := v1admission.AdmissionResponse{
		UID: req.UID,
	}
	// 业务逻辑，判断deployment的副本数是否大于3
	var maxReplicas int32 = 3
	if *deployment.Spec.Replicas > maxReplicas {
		respStructure.Allowed = false
		respStructure.Result = &metav1.Status{
			Message: fmt.Sprintf("Deployment %v has too many replicas", deployment.Name),
			Code:    http.StatusForbidden,
		}
	} else {
		respStructure.Allowed = true
		respStructure.Result = &metav1.Status{
			Code: http.StatusOK,
		}
	}
	admissionResp.Response = &respStructure

	klog.Infof("sending response: %s....", admissionResp.Response.String()[:130])
	respByte, err := json.Marshal(admissionResp)
	if err != nil {
		klog.Errorf("Can't encode response messages: %v", err)
		c.JSON(http.StatusInternalServerError, gin.H{
			"message": fmt.Errorf("can't encode response messages: %v", err).Error(),
		})
		return
	}
	klog.Infof("prepare to write response...")

	// 设置响应头的 Content-Type 为 application/json 并写入响应内容
	c.Data(http.StatusOK, "application/json", respByte)
	klog.Infof("response sent done")
}

func main() {
	r := gin.Default()

	r.GET("/ping", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "pong",
		})
	})
	r.POST("/debug", handleDebugRequest)
	r.POST("/limit-deployment-replicas", handleLimitDeploymentReplicas)
	sCert, err := tls.LoadX509KeyPair(tlsCertFile, tlsKeyFile)
	if err != nil {
		panic(err)
	}
	tlsConfig := &tls.Config{
		Certificates: []tls.Certificate{sCert},
		// TODO: uses mutual tls after we agree on what cert the apiserver should use.
		// ClientAuth:   tls.RequireAndVerifyClientCert,
	}

	s := &http.Server{
		Addr:      ":443",
		Handler:   r,
		TLSConfig: tlsConfig,
	}
	err = s.ListenAndServeTLS("", "")
	if err != nil {
		panic(err)
	}
}
```



https://dev.to/cylon/shen-ru-jie-xi-kubernetes-admission-webhooks-25gh