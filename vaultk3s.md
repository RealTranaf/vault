**Chạy Vault trong môi trường Kubernetes**

Vault được cài đặt trong Kubernetes thông qua package manager Helm. Thực hiện thêm repo và cập nhật:

```
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update
helm search repo hashicorp/vault
```

Nên tạo một namespace riêng cho Helm:

```
kubectl create namespace vault
```

Tạo một file config cho vault như sau: [vaults-value.yaml](vaults-k3s/vault-values.yaml) và thực hiện cài đặt:

```
helm install vault hashicorp/vault -n vault -f vault-values.yaml
```

Sau khi cài đặt, Vault sẽ ở trạng thái khóa (sealed) và cần thực hiện mở khóa. Thực hiện init vault:

```
kubectl exec -n vault -it vault-0 -- vault operator init
```

Lệnh tạo 5 unseal key và initial root token. Lưu các đoạn key này vào một nơi khác và thực hiện unseal:

```
kubectl exec -n vault -it vault-0 -- vault operator unseal
```

Vault sẽ yêu cầu người dùng nhập 3 unseal key để mở khóa Vault. Thực hiện nhập key. Nếu thành công, có thể kiểm tra bằng cách chạy:

```
kubectl exec -n vault -- vault status
```

**Thiết lập secret trong Vault:**

Thực hiện tạo secret:

```
kubectl exec -n vault vault-0 -- vault kv put secret/logging/logging-be \
  spring.datasource.username='root' \
  spring.datasource.password='root'
```

Kiểm tra secret:

```
kubectl exec -n vault vault-0 -- vault kv get secret/logging/logging-be
```

```
kubectl exec -n vault vault-0 -- vault kv get -format=json secret/logging/logging-be
```

```
kubectl exec -n vault vault-0 -- vault kv metadata get secret/logging/logging-be
```

Note: Trong KV v2 thì đường dẫn secret sẽ có /data (secret/data/...) nhưng trong CLI ko cần khai báo cả /data

Địa chỉ Vault nội bộ trong cluster:

```
http://vault.vault.svc.cluster.local:8200
```

Các ứng dụng sẽ kết nối tới Vault qua các service account: **service-account.yaml:**

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: backend
  namespace: logging
```

thực hiện apply, kiểm tra service account của backend:

```
kubectl get serviceaccount -n logging
```

Không nên dùng service account default cho ứng dụng mà nên tạo các account riêng. Kiểm tra service account của Vault:

```
kubectl get pod vault-0 -n vault \
  -o jsonpath='{.spec.serviceAccountName}'
```

trả về **vault**. Thực hiện enable Kubernetes auth:

```
kubectl exec -n vault vault-0 -- \
  vault auth enable kubernetes
```

Cấu hình Vault kết nối tới Kubernetes API:

```
kubectl exec -n vault vault-0 -- \
  vault write auth/kubernetes/config \
  kubernetes_host="https://$KUBERNETES_SERVICE_HOST:$KUBERNETES_SERVICE_PORT"
```
hoặc 
```
kubectl exec -n vault vault-0 -- sh -c '
vault write auth/kubernetes/config \
  kubernetes_host="https://$KUBERNETES_SERVICE_HOST:$KUBERNETES_SERVICE_PORT"
'
```

Kiểm tra:

```
kubectl exec -n vault vault-0 -- \
  vault read auth/kubernetes/config
```

**Cấp quyền TokenReview cho Vault**

Vault cần quyền gọi Kubernetes TokenReview API để xác thực JWT của ứng dụng. Vault yêu cầu ServiceAccount dùng cho token reviewer có quyền system:auth-delegator

**vault-token-review.yaml**

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: vault-token-review
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
subjects:
  - kind: ServiceAccount
    name: vault
    namespace: vault
```

apply, rồi kiểm tra:

```
kubectl get clusterrolebinding vault-token-review
```

**Tạo policy cho logging-be**

Để giới hạn quyền cho secret vừa tạo, cần tạo một policy cho vault. Tạo file sau **logging-be-policy.hcl**:

```
path "secret/data/logging/logging-be" {
  capabilities = ["read"]
}
```

Apply:

```
kubectl cp logging-be-policy.hcl \
  vault/vault-0:/tmp/logging-be-policy.hcl

kubectl exec -n vault vault-0 -- \
  vault policy write logging-be /tmp/logging-be-policy.hcl
```

policy là **logging-be**. Policy này sẽ giới hạn quyền của một service account được tương tác với secret trong vault ntn.

**Tạo Vault Kubernetes role**

Cấu trúc liên kết

```
Kubernetes ServiceAccount
        ↓
   backend (service account của backend)
        ↓
Vault Kubernetes Role
        ↓
    logging-be policy
        ↓
secret/logging/logging-be
```

Tạo role:

```
kubectl exec -n vault vault-0 -- \
  vault write auth/kubernetes/role/logging-be \
  bound_service_account_names=backend \
  bound_service_account_namespaces=logging \
  policies=logging-be \
  ttl=1h
```

- Gắn liên service account của ứng dụng với role là logging-be.
- Role này có policy là cái vừa apply lúc trước.
- Giới hạn quyền hạn cho ứng dụng qua service acount.

Kiểm tra:

```
kubectl exec -n vault vault-0 -- \
  vault read auth/kubernetes/role/logging-be
```

Kết quả cần có:

```
bound_service_account_names                 [backend]
bound_service_account_namespace_selector    n/a
bound_service_account_namespaces            [logging]
policies                                    [logging-be]
```

Như vậy ứng dụng chỉ có thể sử dụng service account

```
ServiceAccount: backend
Namespace: logging
```

mới có thể login và auth vào vault thông qua role.

**Cấu hình lại deployment của ứng dụng**

Thêm service account vào trong deployment:

```
spec:
  template:
    spec:
      serviceAccountName: logging-be
```

Thực hiện thêm các dependency vào trong pom.xml, tương tự như sau:

```
spring.cloud.vault.uri=http://vault.vault.svc.cluster.local:8200
spring.cloud.vault.authentication=KUBERNETES
spring.cloud.vault.kubernetes.role=logging-be

spring.cloud.vault.kv.enabled=true
spring.cloud.vault.kv.backend=secret
spring.cloud.vault.kv.default-context=logging/logging-be
```

- role và path tới secret phải đúng với role và path đã set lúc trước

Note: bỏ biến ra khỏi file properties, ko set kiểu placeholder như spring.datasource.username=${DB_USERNAME} và bỏ biến ra khỏi các file như Dockerfile hay deployment.yaml

Thực hiện build và chạy lại ứng dụng, nếu backend chạy được và lấy được dữ liệu từ DB => thành công.

Luồng xử lý auth của Vault và ứng dụng:

```
Kubernetes Pod: backend
        |
        | ServiceAccount JWT
        v
Vault Kubernetes Authentication
        |
        | Kiểm tra Role: logging-be
        v
Vault cấp token theo Policy
        |
        | Quyền read secret cụ thể
        v
secret/logging/logging-be
        |
        | Nạp datasource properties
        v
Spring Boot
        |
        v
MySQL Service: mysql:3306
``` 