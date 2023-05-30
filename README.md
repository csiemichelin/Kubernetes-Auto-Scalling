# Kubernetes-Auto-Scalling
實現Kubeadm deployment and pod and Service和Kubernetes Auto Scalling
# 雲端K8s AutoScalling實作
## K8s元件
![](https://hackmd.io/_uploads/SyFCbX7Ih.jpg?raw=true)
## 環境
1. VM :
建議使用VMWare，使用Virtual Box有時候會有一些問題
使用了兩台VM，一台當master、一台當worker node(node1)
software : VMware WorkStation 17 player
4 CPU cores
memory : 8GB
OS : ubuntu-18.04-desktop-amd64.iso
2. Host:
OS : Win10
3. Kubernetes version = "1.21.3-00"
## Kubeadm 安裝
### Master & worker node(node1) 都須作設定
#### 更新與安裝
```
sudo rm /var/lib/apt/lists/lock
sudo rm /var/cache/apt/archives/lock
sudo rm /var/lib/dpkg/lock
sudo apt-get remove --purge appstream gnome*
sudo apt update -y
sudo apt upgrade -y
sudo apt install vim net-tools wget -y
```
#### 網路設定
1. 查看master and node1的IP，互相ping看看是否有通
```
ifconfig
ping <node_IP>
```
* 得到master ip : 192.168.157.128 & node1 ip : 192.168.157.129
2. 設定hostname (可取名worker node1、master之類的，方便後面辨識)
```
sudo hostnamectl set-hostname <name>
hostname    //查看
```
3. 編輯hosts檔案，可使用vim或是自己熟悉的編輯軟體
```
sudo vim /etc/hosts
```
* master: 
![](https://hackmd.io/_uploads/Hyz7CmMLh.png)
* node1:
![](https://hackmd.io/_uploads/rykwkNfL2.png)
4. 安裝docker，查看version
```
sudo apt-get install docker.io -y
sudo docker version
```
![](https://hackmd.io/_uploads/Skdzg4MUh.png)
5. 啟動docker並查看狀態
```
sudo systemctl enable docker
sudo systemctl start docker
sudo systemctl status docker
```
![](https://hackmd.io/_uploads/SkiolNML3.png)
6. 關閉swap
```
sudo swapoff -a
top
```
![](https://hackmd.io/_uploads/SkQGW4zI3.png)
#### 安裝kubeadm、kubelet 和 kubectl
```
sudo apt-get update && sudo apt-get install -y apt-transport-https curl

curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF

sudo apt-get update
```
我這邊是選擇安裝指定版本 "1.21.3-00"，如果要安裝新版本有些地方可能會需要大幅度修改，但我使用 "1.21.3-00" 版本跑後面的步驟是可以安裝成功的，新版本目前沒測試過。
```
# 安裝最新版本
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

OR

# 指定安裝版本
## 找到可用的版本 
apt-cache madison kubeadm

## 指定版本
K_VER="<version>"
## ex : K_VER="1.21.3-00"

sudo apt-get install -y kubelet=${K_VER} kubectl=${K_VER} kubeadm=${K_VER}
```
#### 修改docker文件
1. /etc/docker裡面創一個 daemon.json
```
sudo vim /etc/docker/daemon.json
```
2. 加入這段
```
{
"exec-opts":["native.cgroupdriver=systemd"]
}
```
3. 重啟docker
```
sudo systemctl restart docker
sudo systemctl status docker
```
### Master端
1. 初始化master端的參數以及**環境變量**，這段主要是設定kubernetes後面一些元件可以使用的IP範圍，要注意最後有沒有出現 warning，這邊如果出現問題的話可以到上面的 "特殊情況" 第三點看看是不是一樣的問題
```
# 跳過這段
# export KUBECONFIG=/etc/kubernetes/admin.conf
# sudo systemctl daemon-reload
# sudo systemctl restart kubelet

# 執行下面這個
sudo kubeadm init   --pod-network-cidr=10.244.0.0/16 --service-cidr=10.245.0.0/16 --apiserver-advertise-address=<master_IP>
```
2. 最後應該會出現successfully的提示，還有指令kubeadm join…後面的要記錄起來，之後worker node才能透過此token加入叢集中
![](https://hackmd.io/_uploads/BJV60MXU2.png)
```
kubeadm join 192.168.157.128:6443 --token xk4s8y.812h51rzzdmg3n7n \
	--discovery-token-ca-cert-hash sha256:d030dec6b9c544f1a8385c65220c508c8402fae7650a2abb35c2c7d08269ec80
```
3. 查看節點
    * 避免出現 “The connection to the server localhost:8080 was refused - did you specify the right host or port?”，這段在 init 的時候會有提示要執行
    ```
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
    ```
    * 查看節點與狀態
    ```
    sudo systemctl status kubelet
    sudo kubectl get nodes
    ```
    ![](https://hackmd.io/_uploads/rkR51mm82.png)
4. 這邊選擇 flannel 網路元件，也可以選擇其他的網路附加元件，如下圖，如果馬上讀取 node 狀態可能還是會 NotReady 狀態
```
sudo kubectl apply -f     https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
![](https://hackmd.io/_uploads/rk9XgXmL3.png)
5. 需要等待一段時間(3-5 mins)，查看node列表，如果正常就會看到 master 是 Ready 狀態，並且所有的 pods 都會是在 Running 狀態
```
sudo kubectl get nodes
```
* 如果一直顯示"Not Ready"，執行下面那行後重置，直接重做Master的部分
    ```
    sudo kubeadm reset
    ```
![](https://hackmd.io/_uploads/r177ZQQU3.png)
6. 把先前 master 複製的指令 “kubeadm join –token…..”**在 worker node 執行** ，這邊如果出現問題的話可以到下面的 "特殊情況" 第三點看看是不是一樣的問題
```
sudo kubeadm join 192.168.157.128:6443 --token xk4s8y.812h51rzzdmg3n7n \
	--discovery-token-ca-cert-hash sha256:d030dec6b9c544f1a8385c65220c508c8402fae7650a2abb35c2c7d08269ec80
```
![](https://hackmd.io/_uploads/ByEA77QUh.png)
7. **master 端執行**，看有沒有出現 worker node 的資訊，並且 Ready，同樣可能會需要幾分鐘
```
sudo kubectl get nodes
```
![](https://hackmd.io/_uploads/HJrBEQmL2.png)
8. 檢查componentstatuses狀態
```
sudo kubectl get cs
```
![](https://hackmd.io/_uploads/BJu64XX8n.png)
* 如果出現Unhealthy，cd 到/etc/kubernetes/manifests資料夾中，將 kube-controller-manager.yaml 和 kube-scheduler.yaml 這兩個檔案中的 –port=0 註解後重新執行
    ```
    sudo systemctl restart kubelet.service
    ```
    ![](https://hackmd.io/_uploads/BkbrBmXLn.png)
    ![](https://hackmd.io/_uploads/rk2urmmIn.png)
    ![](https://hackmd.io/_uploads/rJ70S7mL3.png)
### Kubeadm 安裝結果
![](https://hackmd.io/_uploads/HkkBIQXI3.png)

## Kubeadm deployment and pod and Service(所有指令皆在master端執行)
1. 建立**一個nginx的deployment**，yaml檔中replicas的2代表要建立兩個pod
```
sudo kubectl apply -f https://k8s.io/examples/application/deployment.yaml
sudo kubectl get pod,deploy
```
![](https://hackmd.io/_uploads/HkX1q7XI3.png)
2. Deployment會監控pod的狀態，當一個pod crash掉時，k8s會自動再新建立一個pod讓總體保持兩個，這邊我選擇直接強制刪除一個pod
![](https://hackmd.io/_uploads/HkuKqmmL2.png)
(上為舊的，下為刪除後自動產生的)
3. 透過 kubectl describe 指令查看pod的Labels(標籤)
```
sudo kubectl describe pod <pod_ID>
```
![](https://hackmd.io/_uploads/r1yI3m7I3.png)
4. 下載並修改yaml讓**Service**抓到要連結的pods後，部屬yaml
```
wget https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/service/networking/nginx-svc.yaml

sudo vim nginx-svc.yaml

sudo kubectl create –f nginx-svc.yaml
```
![](https://hackmd.io/_uploads/H1wuaXmUh.png)
把nginx-svc.yaml修改成:
![](https://hackmd.io/_uploads/HyNl0mXLn.png)
![](https://hackmd.io/_uploads/rJveJVmU3.png)
5. 查看新建立的Service
```
sudo kubectl get service
OR
sudo kubectl get svc
```
![](https://hackmd.io/_uploads/SyEoeV7Un.png)

6. 打開瀏覽器輸入Service 的IP就可以瀏覽剛剛創建成功的服務
![](https://hackmd.io/_uploads/HJD014mLn.png)

## K8s Auto Scalling
### 安裝Metrics Server
[Metrics Server 參考網址](https://github.com/kubernetes-sigs/metrics-server#readme)
如果需要使用Auto Scaling的話就必須要安裝一個可以監控pods、nodes等等所消耗的CPU、Memory量，這邊我使用Metrics Server來監控資源使用量。

我使用k8s "1.21.3-00"版本，需要把Metrics Server網站提供的yaml檔案下載下來(可透過linux的"wget"指令)，或是使用github提供的 components.yaml 做修改也可以。將原本的參數註解之後改成這下面這兩個，如下圖 :
```
wget https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

sudo vim components.yaml

# 新增以下參數並註解掉原本的
- --kubelet-preferred-address-types=InternalIP
- --kubelet-insecure-tls
```
![](https://hackmd.io/_uploads/HJ_rzVQUn.png)

修改完成後再部屬到kubernetes上。
```
sudo kubectl apply -f components.yaml
```
![](https://hackmd.io/_uploads/S1tW7N7Ln.png)
![](https://hackmd.io/_uploads/H1sSXVXUh.png)
### Auto Scalling
[Auto Scaling 實作參考網址](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/)
1. 部署php-apache server
```
sudo kubectl apply -f https://k8s.io/examples/application/php-apache.yaml
```
![](https://hackmd.io/_uploads/rkm4rVQU3.png)
2. 創建 HorizontalPodAutoscaler
```
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
```
![](https://hackmd.io/_uploads/BkN18EX83.png)
如果沒有安裝，那建立的 hpa 可能都會是第一個狀態。安裝成功後 Targets 就會正常的顯示。
![](https://hackmd.io/_uploads/SkVG8EQLn.png)
3. 增加負載為了實現pod Scale Out
客戶端 Pod 中的容器以無限循環運行，向 php-apache 服務發送查詢。
```
# Run this in a separate terminal
# so that the load generation continues and you can carry on with the rest of the steps
kubectl run -i --tty load-generator --rm --image=busybox:1.28 --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache; done"
```
![](https://hackmd.io/_uploads/HyDMSD78h.png)

監控pod和hva
```
sudo kubectl get po,hpa
```
此處，CPU消耗已增加到請求的45%。結果php-apache Deployment被增加為6個副本(pod Scale Out)：
![](https://hackmd.io/_uploads/B1xfwvmLh.png)
4. 停止產生負載為了實現pod Scale In
在master的終端中busybox，通過鍵入終止負載生成<Ctrl>+C

大約一分鐘後，多餘的pod會terminate，剩下一個php-apache pod：
![](https://hackmd.io/_uploads/HyY15PQL3.png)

    
## 特殊情況 (過程有問題再看)
💡如果重開機有問題，操作完需要等一下，我通常用上面那個，master、worker node都需要執行，過一段時間在master端 "kubectl get nodes" 看是否成功 Ready
```
sudo swapoff -a
sudo strace -eopenat kubectl version

OR

sudo systemctl restart docker
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```
💡 如果忘記Master 的 join token，也可以直接 "kubeadm reset" 後重跑 initial 一次產生新的 token
```
kubeadm token generate
kubeadm token create <generation_token> --print-join-command --ttl=0
```
💡 如果在 init 的時候出現下圖 WARNING 的問題，可以參考下面的連結解決，主要應該是 docker driver 設定的問題
[參考連結](https://cloud.tencent.com/developer/article/1815028)
```
CentOS -> /usr/lib/systemd/system/docker.service
Ubuntu -> /lib/systemd/system/docker.service
```
![](https://hackmd.io/_uploads/BkyiMQXU2.png)
