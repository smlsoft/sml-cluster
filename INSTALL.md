# คู่มือ Deploy AIS K8s Pre POC Lab

คู่มือนี้เริ่มจาก deploy YAML ลง Kubernetes ที่เตรียมไว้แล้ว โดยรันคำสั่งจาก root ของ repository และใช้ namespace `smlerp` ตาม manifest ปัจจุบัน

ลำดับการทำงาน: **ตรวจ requirements → Namespace → PVC → ไฟล์ config → Secrets → ERP และ Service → Stock Process → ทดสอบด้วย port-forward** ส่วน pgAdmin, LoadBalancer และ Ingress เป็นทางเลือกท้ายเอกสาร

## Requirements ก่อนเริ่ม

| สิ่งที่ต้องมี | เงื่อนไขที่ต้องพร้อม |
| --- | --- |
| Kubernetes Cluster | Node อยู่ในสถานะ `Ready` และมีทรัพยากรพอสำหรับ workload; คู่มือนี้ไม่ครอบคลุมการติดตั้ง cluster หรือ Colima |
| เครื่องที่ใช้ deploy | มี repository นี้, `kubectl`, kubeconfig ของ cluster เป้าหมาย และ `curl` สำหรับทดสอบ HTTP |
| สิทธิ์ Kubernetes | สร้าง Namespace และจัดการ PVC, Pod, Secret, Deployment, Service รวมถึงใช้งาน exec และ port-forward ได้ |
| NFS Storage | มี NFS server, provisioner และ StorageClass ชื่อ `nfs-client` ที่ใช้งานได้ รองรับ `ReadWriteMany` และพื้นที่อย่างน้อย `1Gi` สำหรับ PVC config; node ต้อง mount NFS ได้ |
| PostgreSQL | มีฐานข้อมูลของระบบ พร้อม schema/ข้อมูลตั้งต้นและบัญชีที่แอปต้องใช้; Pod ต้องเข้าถึง host/port ของฐานข้อมูลได้ |
| ระบบคิว | มี broker ที่รองรับ STOMP พร้อม endpoint, port, username, password และ queue/topic ที่ ERP และ Stock Process ใช้ร่วมกันได้; Secret ปัจจุบันระบุ port `61613` ให้ตรวจค่าจริงกับผู้ดูแล |
| ไฟล์ config ของแอป | ได้รับชุดไฟล์ที่ตรงกับ image จากผู้ดูแลแอป รวมค่าการเชื่อมต่อฐานข้อมูล และพร้อมนำไปวางใน PVC ซึ่งทั้งสองบริการ mount ที่ `/etc/config` |
| Container images | Node ดึง `ghcr.io/smlsoft/smljavawebservice:3.9.13` และ `ghcr.io/smlsoft/smljavawebservice:3.9.14-jdk11` ได้; หาก registry ต้องยืนยันตัวตน ให้เตรียม image pull Secret และเพิ่ม `imagePullSecrets` ใน Deployment ก่อน deploy |
| เครือข่าย | Pod resolve DNS และเชื่อมต่อฐานข้อมูล/คิวได้ รวมถึง node เชื่อมต่อ NFS และ registry ได้; ใช้ endpoint ที่เข้าถึงได้จาก cluster เพราะ `localhost` ภายใน Pod หมายถึง Pod นั้นเอง |

Repository ไม่มีชุดไฟล์ config ของแอปหรือขั้นตอนสร้าง schema/restore ฐานข้อมูล จึงต้องเตรียมสองส่วนนี้จากผู้ดูแลก่อนเริ่ม เอกสารนี้ไม่กำหนดชื่อไฟล์หรือรูปแบบ config ขึ้นเอง

ไฟล์ `postgresql/postgresql.yaml` และ `rabbitmq/rabbit-mq.yaml` เป็น **Docker Compose** ไม่ใช่ Kubernetes manifest จึงไม่ต้อง `kubectl apply` สองไฟล์นี้ และ RabbitMQ ตัวอย่างยังไม่ได้ตั้งค่า STOMP plugin สำหรับแทน broker ที่ใช้อยู่

หาก NFS ยังไม่พร้อม ดูข้อมูลประกอบใน [COLIMA-NFS-SETTING.md](COLIMA-NFS-SETTING.md) โดยปรับ IP, export path และ network ให้ตรงกับระบบจริง

## 1. ตรวจ cluster เป้าหมายและสร้าง Namespace

ตรวจ context ให้ตรงกับ cluster ที่ต้องการก่อนสร้าง resource:

```bash
kubectl config current-context
kubectl cluster-info
kubectl get nodes
kubectl get storageclass nfs-client
```

Node ต้อง `Ready` และต้องพบ StorageClass `nfs-client` จากนั้นสร้าง namespace ด้วย YAML ที่ส่งผ่าน stdin:

```bash
kubectl create namespace smlerp --dry-run=client -o yaml | kubectl apply -f -
```

ใช้ `-n smlerp` ในคำสั่งถัดไปให้ตรงกับ `metadata.namespace` ในไฟล์

## 2. สร้าง PVC สำหรับ config ที่ทั้งสองบริการใช้ร่วมกัน

```bash
kubectl apply -f smlerp-service/smlerp-service-config-pvc.yaml
kubectl wait -n smlerp --for=jsonpath='{.status.phase}'=Bound pvc/smlerp-config-pvc --timeout=180s
kubectl get pvc -n smlerp
```

PVC `smlerp-config-pvc` ต้องเป็น `Bound` ก่อนทำต่อ โดยขอพื้นที่ `1Gi` และใช้ `ReadWriteMany` เพื่อให้ ERP กับ Stock Process mount ร่วมกันได้

หาก PVC ยัง `Pending` ให้ตรวจ provisioner, NFS และ Events ก่อน deploy แอป:

```bash
kubectl describe pvc smlerp-config-pvc -n smlerp
```

## 3. นำไฟล์ config ลง PVC ก่อนเริ่มแอป

สร้าง Pod ชั่วคราวเพื่อ mount PVC และคัดลอกไฟล์จากเครื่องที่ deploy ตัวอย่างนี้ต้องดึง image `busybox:1.37` ได้ด้วย:

```bash
kubectl apply -f - <<'YAML'
apiVersion: v1
kind: Pod
metadata:
  name: smlerp-config-loader
  namespace: smlerp
spec:
  restartPolicy: Never
  containers:
    - name: config-loader
      image: busybox:1.37
      command: ["sh", "-c", "sleep 3600"]
      volumeMounts:
        - name: config
          mountPath: /etc/config
  volumes:
    - name: config
      persistentVolumeClaim:
        claimName: smlerp-config-pvc
YAML

kubectl wait -n smlerp --for=condition=Ready pod/smlerp-config-loader --timeout=180s
```

เปลี่ยน `/absolute/path/to/app-config` เป็น directory ที่มีไฟล์ config จริง รักษาชื่อไฟล์และโครงสร้าง directory ตามที่แอปกำหนด:

```bash
kubectl cp /absolute/path/to/app-config/. smlerp/smlerp-config-loader:/etc/config
kubectl exec -n smlerp smlerp-config-loader -- ls -la /etc/config
kubectl delete pod smlerp-config-loader -n smlerp
```

ตรวจว่าไฟล์ครบและมีสิทธิ์อ่านสำหรับ user ที่แอปใช้ การลบ Pod ชั่วคราวไม่ลบไฟล์ใน PVC หากมีไฟล์อยู่แล้ว ให้ตรวจให้แน่ใจว่าการคัดลอกจะไม่ทับ config ที่ยังใช้งานอยู่

## 4. เตรียมและสร้าง Secrets ของทั้งสองบริการ

ทำสำเนา Secret manifests ไว้นอก repository เพื่อใส่ค่าของ environment เป้าหมาย โดยใช้ terminal เดิมสำหรับคำสั่งที่อ้างตัวแปร `lab_manifest_dir`:

```bash
lab_manifest_dir=$(mktemp -d)
chmod 700 "$lab_manifest_dir"
cp smlerp-service/smlerp-service-secret.yaml "$lab_manifest_dir/erp-secret.yaml"
cp stockprocess-service/smlerp-stock-process-service-secret.yaml "$lab_manifest_dir/stock-secret.yaml"
chmod 600 "$lab_manifest_dir/erp-secret.yaml" "$lab_manifest_dir/stock-secret.yaml"
```

เปิดแก้สองไฟล์ใน directory นี้ด้วย editor ที่ใช้งาน เปลี่ยนค่า endpoint และ credentials ให้ตรงกับ broker จริง โดยคงชื่อ Secret และ namespace เดิมไว้

| ค่าที่ต้องตรวจ | ERP Secret | Stock Process Secret |
| --- | --- | --- |
| Directory ของ config | `SERVICE_CONFIG_DIR` = `/etc/config` | `SERVICE_CONFIG_DIR` = `/etc/config` |
| Protocol/provider | `SML_NEXT_PROCESS_SERVER_MODE` = `STOMP` | `STOCK_PROCESS_CONSUMER_PROVIDER` = `STOMP` |
| Broker host | `SML_NEXT_PROCESS_SERVER` | `STOCK_PROCESS_CONSUMER_PROVIDER_SERVER` |
| Broker port | `SML_NEXT_PROCESS_SERVER_PORT` | `STOCK_PROCESS_CONSUMER_PORT` |
| Username | `SML_NEXT_PROCESS_USERNAME` | `STOCK_PROCESS_CONSUMER_USERNAME` |
| Password | `SML_NEXT_PROCESS_PASSWORD` | `STOCK_PROCESS_CONSUMER_PASSWORD` |
| Queue/topic ปลายทาง | `SML_NEXT_PROCESS_CHANNEL_NAME` | `STOCK_PROCESS_CONSUMER_TOPIC` |

ไฟล์เดิมใช้ `data:` ซึ่งค่าต้องเป็น base64 หากต้องการกรอกข้อความโดยตรง ให้เปลี่ยนทั้ง section เป็น `stringData:` และใส่ค่าจริงของ **ทุก key** เป็น string เช่น `"61613"` โดยไม่เก็บ section `data:` เดิมไว้ด้วย อย่านำค่า base64 เดิมมาใส่ใน `stringData:`

เมื่อแก้ครบแล้ว:

```bash
kubectl apply -f "$lab_manifest_dir/erp-secret.yaml"
kubectl apply -f "$lab_manifest_dir/stock-secret.yaml"
kubectl get secret smlerp-service-secret smlerp-stock-process-service-secret -n smlerp
```

เก็บไฟล์ที่แก้ไว้ในที่จัดเก็บ credentials ที่เหมาะสม และลบสำเนาชั่วคราวเมื่อเลิกใช้ ไม่ commit credentials ของ environment ลง repository

## 5. Deploy ERP และสร้าง Service แบบ ClusterIP

เมื่อ PVC มี config และ Secrets พร้อมแล้ว จึงเริ่ม ERP:

```bash
kubectl apply -f smlerp-service/smlerp-service-service-clusterip.yaml
kubectl apply -f smlerp-service/smlerp-service-deployment.yaml
kubectl rollout status deployment/smlerp-service -n smlerp --timeout=300s
kubectl get pods,svc -n smlerp
```

Deployment ใช้ image `3.9.13` และ Service `smlerp-service` เปิด port `8080` ภายใน cluster ต้องรอ rollout สำเร็จและ Pod เป็น `1/1 Running` ก่อนทดสอบต่อ

## 6. Deploy Stock Process

```bash
kubectl apply -f stockprocess-service/smlerp-stock-process-service-deployment.yaml
kubectl rollout status deployment/smlerp-stock-process-service -n smlerp --timeout=300s
kubectl get pods -n smlerp
```

Deployment นี้ใช้ image `3.9.14-jdk11` และ PVC เดียวกับ ERP แต่ใช้ Secret ของตัวเอง ไม่ต้องสร้าง Service สำหรับขั้นตอน consume งานจาก broker ตรวจ log ว่าเชื่อมต่อ broker และปลายทางที่กำหนดได้:

```bash
kubectl logs -n smlerp deployment/smlerp-stock-process-service --tail=100
```

## 7. ทดสอบผ่าน port-forward

เปิด terminal แรกและปล่อยคำสั่งนี้ทำงานไว้:

```bash
kubectl port-forward -n smlerp service/smlerp-service 8080:8080
```

เปิดอีก terminal แล้วเรียก health endpoint ตาม Deployment โดยใช้ตัวพิมพ์เล็ก/ใหญ่ให้ตรง:

```bash
curl --fail --show-error http://127.0.0.1:8080/SMLJavaWebService/health
```

ต้องได้ HTTP สำเร็จ สำหรับ Stock Process ซึ่งไม่มี Service ให้เปิด port-forward ไปยัง Deployment ในอีก terminal:

```bash
kubectl port-forward -n smlerp deployment/smlerp-stock-process-service 8081:8080
```

```bash
curl --fail --show-error http://127.0.0.1:8081/SMLJavaWebService/health
```

port-forward ใช้ระหว่างที่คำสั่งทำงานเท่านั้น หยุดด้วย `Ctrl+C` หาก port `8080` บนเครื่องถูกใช้แล้ว ให้เปลี่ยนฝั่ง local เช่น `18080:8080` และเรียก URL ที่ port `18080`

Health สำเร็จยืนยันว่าแอปตอบ endpoint ได้ แต่ยังต้องทดสอบงานจริงที่เข้าถึงฐานข้อมูล ส่งข้อความจาก ERP และให้ Stock Process consume สำเร็จ จึงจะยืนยันการทำงานครบระบบได้

## 8. ทางเลือก: Deploy pgAdmin

pgAdmin เป็นเครื่องมือจัดการฐานข้อมูล ไม่ใช่ dependency สำหรับเริ่ม ERP ก่อน deploy ให้ปรับ `PGADMIN_DEFAULT_EMAIL` และ `PGADMIN_DEFAULT_PASSWORD` ใน `pgadmin/pgadmin-deployment.yaml` ให้เหมาะกับ environment

```bash
kubectl apply -f pgadmin/pgadmin-service.yaml
kubectl apply -f pgadmin/pgadmin-deployment.yaml
kubectl rollout status deployment/pgadmin -n smlerp --timeout=300s
kubectl port-forward -n smlerp service/pgadmin 5050:80
```

เปิด [pgAdmin](http://127.0.0.1:5050/pgadmin4/) ตามค่า `SCRIPT_NAME=/pgadmin4` ใน Deployment แล้วเพิ่ม PostgreSQL server ด้วย endpoint ที่เข้าถึงได้จาก Pod นี้

Manifest ปัจจุบันไม่มี volume สำหรับข้อมูล pgAdmin ดังนั้นการลงทะเบียน server/การตั้งค่าใน pgAdmin อาจหายเมื่อ Pod ถูกสร้างใหม่

## 9. ทางเลือก: เปิดบริการผ่าน LoadBalancer หรือ Ingress

### LoadBalancer

ต้องมีระบบจัดสรร external IP ที่พร้อมใช้งานใน cluster จากนั้นใช้ manifest นี้แทน Service แบบ ClusterIP:

```bash
kubectl apply -f smlerp-service/smlerp-service-loadbalancer-service.yaml
kubectl get service smlerp-service -n smlerp
```

ทั้งสองไฟล์ Service ใช้ชื่อ `smlerp-service` เดียวกัน การ apply นี้จึงเปลี่ยน Service เดิมเป็น `LoadBalancer` ไม่ได้สร้าง Service เพิ่ม อย่า apply ทั้งสองไฟล์สลับกัน เมื่อได้ external IP แล้วทดสอบ `http://<EXTERNAL-IP>:8080/SMLJavaWebService/health` หาก IP เป็น `Pending` ให้ตรวจระบบ LoadBalancer ของ cluster

### Ingress

ต้องมี Ingress controller ที่รองรับ `ingressClassName: nginx` ใน cluster ก่อน และจัดเตรียมเส้นทางเครือข่าย/DNS สำหรับผู้เรียกใช้งาน โดยใช้ ERP Service แบบ ClusterIP จากขั้นตอน 5 ได้

**แก้ `smlerp-service/smlerp-service-ingress.yaml` ก่อน apply** เพราะค่าปัจจุบันไม่ตรงกับ Service/health path:

- เปลี่ยน backend `service.port.number` จาก `80` เป็น `8080`
- เปลี่ยน path จาก `/SMLJavawebService(/|$)(.*)` เป็น `/SMLJavaWebService` และคง `pathType: Prefix` เพื่อใช้ prefix ปกติให้ตรงกับ path ของแอป

```bash
kubectl apply -f smlerp-service/smlerp-service-ingress.yaml
kubectl get ingress -n smlerp
```

ทดสอบผ่าน address ของ controller ที่ path `/SMLJavaWebService/health` ส่วน `pgadmin/pgadmin-ingress.yaml` เป็นตัวเลือกแยกที่ใช้ rewrite และ configuration snippet เฉพาะ controller ต้องตรวจนโยบาย controller และความสอดคล้องของ path `/pgadmin` กับ `SCRIPT_NAME=/pgadmin4` ก่อนใช้งาน; คู่มือหลักใช้ port-forward สำหรับ pgAdmin

## ตรวจปัญหาเมื่อ deploy ไม่สำเร็จ

```bash
kubectl get pods,pvc,svc -n smlerp
kubectl get events -n smlerp --sort-by=.metadata.creationTimestamp
kubectl describe deployment smlerp-service -n smlerp
kubectl logs -n smlerp deployment/smlerp-service --tail=100
```

| อาการ | สิ่งที่ควรตรวจ |
| --- | --- |
| PVC `Pending` / mount ไม่สำเร็จ | StorageClass, NFS provisioner, export path, network และสิทธิ์บน NFS; ใช้ `kubectl describe pvc` และ `kubectl describe pod <pod-name> -n smlerp` |
| `CreateContainerConfigError` | มี Secret ชื่อตรงกับ Deployment ใน namespace เดียวกันหรือไม่ |
| `ImagePullBackOff` | ชื่อ image/tag, registry credentials, network และ architecture ที่ image รองรับ |
| `CrashLoopBackOff` / Pod ไม่ Ready | ไฟล์ใน `/etc/config`, สิทธิ์อ่าน, การเชื่อมต่อฐานข้อมูล/คิว และ logs; ใช้ `kubectl logs <pod-name> -n smlerp --previous` หาก container restart แล้ว |
| port-forward หา Pod พร้อมใช้ไม่ได้ | Deployment rollout สำเร็จหรือไม่ และ selector ของ Service ตรงกับ Pod หรือไม่ |
| Ingress ตอบ 404/502/503 | IngressClass/controller, path ที่ตัวพิมพ์ตรงกัน, backend port `8080` และสถานะ readiness ของ ERP |

หากแก้ Secret หลัง deploy แล้ว ให้ restart Deployment ที่เกี่ยวข้องเพื่อโหลด environment ใหม่ เช่น:

```bash
kubectl rollout restart deployment/smlerp-service -n smlerp
kubectl rollout status deployment/smlerp-service -n smlerp --timeout=300s
```

สำหรับ Stock Process ใช้ชื่อ `smlerp-stock-process-service` แทน เมื่อแก้ไฟล์ config บน PVC ให้ตรวจวิธี reload ของแอป หรือ restart บริการที่เกี่ยวข้องหากต้องโหลด config ตอนเริ่มต้น

## เอกสารอ้างอิง

- [Kubernetes: Persistent Volumes และ PVC](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Kubernetes: การใช้ port-forward](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/)
- [Kubernetes: Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
