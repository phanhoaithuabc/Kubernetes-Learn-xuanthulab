# Learn Kubernetes

prequistist
virtualbox

chmode +x install-docker-kube.sh
Tại thư mục kubernetes-centos7/master/ gõ lệnh vagrant để tạo máy master.xtl
vagrant up

Gõ lệnh sau để khở tạo là nút master của Cluster
kubeadm init --apiserver-advertise-address=172.16.10.100 --pod-network-cidr=192.168.0.0/16

Sau khi lệnh chạy xong, chạy tiếp cụm lệnh nó yêu cầu chạy sau khi khởi tạo- để chép file cấu hình đảm bảo trình kubectl trên máy này kết nối Cluster
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

Tiếp đó, nó yêu cầu cài đặt một Plugin mạng trong các Plugin tại addon, ở đây đã chọn calico, nên chạy lệnh sau để cài nó
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.1/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.1/manifests/custom-resources.yaml

Cấu hình kubectl máy trạm truy cập đến các Cluster
để xem nội dung cấu hình kubectl gõ lệnh
kubectl config view

Giờ bạn sẽ thực hiện kết hợp 2 file: config và config-mycluster thành 1 và lưu trở lại config.
export KUBECONFIG=~/.kube/config:~/.kube/config-mycluster-1
kubectl config view --flatten > ~/.kube/config_temp
mv ~/.kube/config_temp ~/.kube/config

# Các ngữ cảnh hiện có trong config
kubectl config get-contexts

Ký hiệu * là cho biết context hiện tại, nếu muốn chuyển làm việc sang context có tên kubernetes-admin@kubernetes (nối với cluster mới tạo ở trên) thì gõ lệnh
kubectl config use-context kubernetes-admin@kubernetes

Kết nối Node vào Cluster
kubeadm token create --print-join-command

# node worker kết nối vào Cluster
kubeadm join 172.16.10.100:6443 --token 5ajhhs.atikwelbpr0 ...

# Lấy thông tin Cluster
kubectl cluster-info

# Các Node có trong Cluster
kubectl get nodes
kubectl get node -o wide

# Muốn lấy manifest mô tả thông tin tài nguyên
kubectl get node -o yaml
kubectl get pod/ungdung1 -o yaml

# Các pod đang chạy trong tất cả các namespace
kubectl get pods -A

# describe node
kubectl describe node/worker1.kube.node
node master thì có lable là: node-role.kubernetes.io/master=

# gán nhãn cho 1 node:
kubectl label node worker.kube.node nodeabc=ungdungpython
lấy ra node có 1 nhãn mà mình chỉ định:
kubectl get node -l "nodeabc=ungdungpython"
xóa 1 label: kubectl label node worker1.kube.node nodeabc-

# Create dashboard
Do cấu hình mặc định của Kubernetes Cluster, cổng được mở ra ngoài phải chọn trong khoảng 30000-32767 sau này bạn có thể sửa cấu hình này với tham số chạy Cluster --service-node-portrange

# apply dashboard
kubectl apply -f dashboard.yaml

# check pod trong namespace
kubectl get pod -n kubernetes-dashboard

Tạo kubernetes-dashboard-certs, xác thực SSL
Ta sẽ dùng OpenSSL để sinh certificates tự xác thực SSL (ngoài ra khi triển khai thực tế có thể lấy các certificate do mua hoặc miễn phí từ https://letsencrypt.org/ theo Domain), chạy các lệnh sau:
sudo mkdir certs
sudo chmod 777 -R certs
openssl req -nodes -newkey rsa:2048 -keyout certs/dashboard.key -out certs/dashboard.csr -subj "/C=/ST=/L=/O=/OU=/CN=kubernetes-dashboard"
openssl x509 -req -sha256 -days 365 -in certs/dashboard.csr -signkey certs/dashboard.key -out certs/dashboard.crt
sudo chmod -R 777 certs

Trên Windows nếu không có lệnh openssl có thể tạo một Image có cài OpenSSL ví dụ Dockerfile sau:
FROM alpine

RUN apk update && \
  apk add --no-cache openssl && \
  rm -rf /var/cache/apk/*

WORKDIR /

ENTRYPOINT ["openssl"]

Build image đặt tên ví dụ, myopenssl
docker build -t myopenssl -f Dockerfile .

Sau đó phát sinh cert trên Windows
mkdir certs
docker run --rm -v ${PWD}/certs:/certs/ myopenssl req -nodes -newkey rsa:2048 -keyout /certs/dashboard.key -out /certs/dashboard.csr -subj "/C=/ST=/L=/O=/OU=/CN=kubernetes-dashboard"
docker run --rm -v ${PWD}/certs:/certs/ myopenssl x509 -req -sha256 -days 365 -in /certs/dashboard.csr -signkey /certs/dashboard.key -out /certs/dashboard.crt

Sau khi hoàn thành thì các thông tin certificate lưu ở thư mục certs, chạy lệnh sau để tạo Secret
kubectl create secret generic kubernetes-dashboard-certs --from-file=certs -n kubernetes-dashboard

xem secret kubernetes-dashboard
kubectl get secret -n kubernetes-dashboard
kubectl describe secret kubernetes-dashboard-certs -n kubernetes-dashboard

Giờ truy cập địa chỉ https://192.168.10.100:31000 sẽ vào màn hình đăng nhập

Sau đó chạy lệnh:
kubectl apply -f admin-user.yaml

Tiếp theo chạy lệnh sau để lấy Token
kubectl get secret -n kubernetes-dashboard 
kubectl describe secret/admin-user-token -n kubernetes-dashboard

Copy toàn bộ đoạn Token để đưa vào đăng nhập

# với docker desktop
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml 
kubectl proxy
kubectl apply -f create-service-cccount.yaml
kubectl apply -f create-cluster-role-binding.yaml
kubectl -n kubernetes-dashboard create token admin-user


các lệnh pod
kubectl get pods	Liệt kê các POD trong namespace hiện tại, thêm tham số -o wide hiện thị chi tiết hơn, thêm -A hiện thị tất cả namespace, thêm -n namespacename hiện thị Pod của namespace namespacename
kubectl explain pod --recursive=true	Xem cấu trúc mẫu định nghĩa POD trong file cấu hình yaml
kubectl apply -f firstpod.yaml	Triển khai tạo các tài nguyên định nghĩa trong file firstpod.yaml
kubectl delete -f firstpod.yaml	Xóa các tài nguyên tạo ra từ định nghĩa firstpod.yaml
kubectl describe pod/namepod	Lấy thông tin chi tiết POD có tên namepod, nếu POD trong namespace khác mặc định thêm vào tham số -n namespace-name
kubectl logs pod/podname	Xem logs của POD có tên podname

kubectl exec mypod command
Chạy lệnh từ container của POD có tên mypod, nếu POD có nhiều container thêm vào tham số -c và tên container, mặc định sẽ vào container thứ 1
vd: kubectl exec ungdung1 ls /
    kubectl exec -it tools bash

kubectl exec -it mypod bash	Chạy lệnh bash của container trong POD mypod và gắn terminal
kubectl proxy	Tạo server proxy truy cập đến các tài nguyên của Cluster. http://localhost/api/v1/namespaces/default/pods/mypod:8085/proxy/, truy cập đến container có tên mypod trong namespace mặc định.
kubectl delete pod/mypod	Xóa POD có tên mypod

Triển khai tạo Pod từ file này, thực hiện lệnh sau
kubectl apply -f 1-swarmtest-node.yaml
kubectl apply -f 2-nginx.yaml
kubectl apply -f 3-tools.yaml
kubectl apply -f 4-nginx-swamtest.yaml
kubectl apply -f 5-nginx-swamtest-vol.yaml
kubectl delete -f example/pods (delete all pod)

Mặc định Kubernetes không tạo và chạy POD ở Node Master để đảm bảo yêu cầu an toàn, nếu vẫn muốn chạy POD ở Master thi hành lệnh sau:
kubectl taint nodes --all node-role.kubernetes.io/master-

# xem các sự kiện xảy ra trong cluster
kubectl get event/ev

# chỉnh sửa nôi dung của file manifest của pod:
kubectl edit po/ungdung1 

Truy cập Pod từ bên ngoài Cluster (Kiểm tra - Debug)
Trong thông tin của Pod ta thấy có IP của Pod và cổng lắng nghe, tuy nhiên Ip này là nội bộ, chỉ các Pod trong Cluster liên lạc với nhau. Nếu bên ngoài muốn truy cập cần tạo một Service để chuyển traffic bên ngoài vào Pod (tìm hiểu sau), tại đây để debug - truy cập kiểm tra bằng cách chạy proxy:
kubectl proxy
Hoặc
kubectl proxy --address="0.0.0.0" --accept-hosts='^*$'
Truy cập đến địa chỉ http://localhost/api/v1/namespaces/default/pods/mypod:8085/proxy/
Khi kiểm tra chạy thử, cũng có thể chuyển cổng để truy cập. Ví dụ cổng host 8080 được chuyển hướng truy cập đến cổng 8085 của POD mypod
kubectl port-forward mypod 8080:8085

Ổ đĩa / Volume trong POD
Để thêm các Volume đầu tiên cần định nghĩa ổ đĩa ở trường spec.volumes. Tại mỗi container gắn ổ đĩa vào nếu cần với thuộc tính volumeMounts
Nếu muốn sử dụng ổ đĩa - giống nhau về dữ liệu trên nhiều POD, kể cả các POD đó chạy trên các máy khác nhau thì cần dùng các loại đĩa Remote - ví dụ NFS

Cấu hình thăm dò Container còn sống
Bạn có thể cấu hình livenessProbe cho mỗi container, để Kubernetes kiểm tra xem container còn sống không. Ví dụ, đường dẫn kiểm tra là /healthycheck, nếu nó trả về mã header trong khoảng 200 đến 400 được coi là sống (tất nhiên bạn cần viết ứng dụng trả về mã này). Trong đó cứ 10s kiểm tra một lần
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
  labels:
    app: mypod
spec:
  containers:
  - name: mycontainer
    image: ichte/swarmtest:php
    ports:
    - containerPort: 8085
    resources: {}

    livenessProbe:
      httpGet:
        path: /healthycheck
        port: 8085
      initialDelaySeconds: 10
      periodSeconds: 10
```

ReplicaSet là một điều khiển Controller - nó đảm bảo ổn định các nhân bản (số lượng và tình trạng của POD, replica) khi đang chạy.
Khi định nghĩa một ReplicaSet (định nghĩa trong file .yaml) gồm các trường thông tin, gồm có trường selector để chọn ra các các Pod theo label, từ đó nó biết được các Pod nó cần quản lý(số lượng POD có đủ, tình trạng các POD). Trong nó nó cũng định nghĩa dữ liệu về Pod trong spec template, để nếu cần tạo Pod mới nó sẽ tạo từ template đó. Khi ReplicaSet tạo, chạy, cập nhật nó sẽ thực hiện tạo / xóa POD với số lượng cần thiết trong khai báo (repilcas).

# Launch các replicaSet
kubectl apply -f 1.rs.yaml

Cấu hình trong ReplicaSet/1.app.yaml định nghĩa một ReplicaSet đặt tên là rsapp, nó quản lý nhân bản 3 POD có nhãn app=rsapp, POD có một container từ image ichte/swarmtest:node
Để lấy các ReplicaSet thực hiện lệnh
kubectl get rs

Thông tin về ReplicaSet có tên rsapp
kubectl describe rs/rsapp
xuất manifest của replicaset
kubectl get rs -o yaml

# monitor component tạo ra every 1s:
watch -n 1 kubectl get all -o wide

Horizontal Pod Autoscaler là chế độ tự động scale (nhân bản POD) dựa vào mức độ hoạt động của CPU đối với POD, nếu một POD quá tải - nó có thể nhân bản thêm POD khác và ngược lại - số nhân bản dao động trong khoảng min, max cấu hình

Ví dụ, với ReplicaSet rsapp trên đang thực hiện nhân bản có định 3 POD (replicas), nếu muốn có thể tạo ra một HPA để tự động scale (tăng giảm POD) theo mức độ đang làm việc CPU, có thể dùng lệnh sau:

kubectl autoscale rs rsapp --max=2 --min=1
Lệnh trên tạo ra một hpa có tên rsapp, có dùng tham chiếu đến ReplicaSet có tên rsapp để scale các POD với thiết lập min, max các POD

Để liệt kê các hpa gõ lệnh
kubectl get hpa -o wide

Để linh loạt và quy chuẩn, nên tạo ra HPA (HorizontalPodAutoscaler) từ cấu hình file yaml
kubectl apply -f 2.hpa.yaml


