# 前置作業
- 下載 gcloud
- 取得SSH Key(執行一次就好)
    ```
    ssh-keygen
    ```  
- 找到id_rsa.pub並增加名稱
![alt text](pic\image-3.png)
# 環境變數  
```
export EMAIL=your_account@smartsurgerytek.com
export GCP_REGION=asia-east1
export GCP_ZONE="${GCP_REGION}-a"
export YOUR_NAME=name
export INSTANCE_NAME="${YOUR_NAME}-cvat-${GCP_ZONE}"
export PUBLIC_KEY_PATH=/path/to/your/public/key.pub
export GCP_SUBDOMAIN=your_subdomain
export CVAT_HOST="${GCP_SUBDOMAIN}.smartsurgerytek.net"
```
- your_account：改成自己的公司信箱 ex. lianchia.lin@smartsurgerytek.com
- YOUR_NAME：改成自己的名稱 ex.lianchia
- PUBLIC_KEY_PATH：改成自己的SSH Key的公開金鑰路徑 ex. /HOME/lianchia/.ssh/id_rsa.pub
![alt text](pic\image.png)
- GCP_SUBDOMAIN：改成自己的網域 ex. dev8
```
export EMAIL=lianchia.lin@smartsurgerytek.com
export GCP_REGION=asia-east1
export GCP_ZONE="${GCP_REGION}-a"
export YOUR_NAME=lianchia
export INSTANCE_NAME="${YOUR_NAME}-cvat-${GCP_ZONE}"
export PUBLIC_KEY_PATH=/home/lianchia/.ssh/id_rsa.pub
export GCP_SUBDOMAIN=dev8
export CVAT_HOST="${GCP_SUBDOMAIN}.smartsurgerytek.net"
```

# 找sandbox
```
gcloud config get-value project
```
正常會顯示：sandbox-446907  
如果沒有可以打  
```
gcloud config set project project_you_want
```
- 有關gcloud的指令都是在本地端執行(DESKTOP)

# 創建新的機器
```
gcloud compute instances create $INSTANCE_NAME \
  --image-family=ubuntu-2204-lts \
  --image-project=ubuntu-os-cloud \
  --machine-type=e2-custom-4-32768 \
  --boot-disk-size=200GB \
  --boot-disk-type=pd-balanced \
  --zone=${GCP_ZONE} \
  --tags=cvat-server-8080 \
  --metadata-from-file ssh-keys=$PUBLIC_KEY_PATH
```
![alt text](pic\image-4.png)
# 尋找機器
```
gcloud compute instances list
```
# 設定IP
```
export EXTERNAL_IP=xxx.xxx.xxx.xxx
```
我的是104.199.202.74  
所以改成：export EXTERNAL_IP=104.199.202.74

# 創建靜態IP
```
gcloud compute addresses create $INSTANCE_NAME-ip --addresses $EXTERNAL_IP --region asia-east1
```
![alt text](pic\image-5.png)
- $INSTANCE_NAME在最一開始的環境變數設定過了
- $EXTERNAL_IP也是前一部設定的

# 設定GCP
```
gcloud dns record-sets transaction start 
--zone=smartsurgerytek-net
```
- 開始設定DNS
```
gcloud dns record-sets transaction add $EXTERNAL_IP \
  --name="${GCP_SUBDOMAIN}.smartsurgerytek.net." \
  --ttl=300 \
  --type=A \
  --zone=smartsurgerytek-net
```
- 設定DNS
```
gcloud dns record-sets transaction execute --zone=smartsurgerytek-net
```
- 執行設定

# 開機
```
gcloud compute ssh --zone $GCP_ZONE $INSTANCE_NAME  -- "export CVAT_HOST=${CVAT_HOST} && exec bash -l"
```
![alt text](pic\image-6.png)

# 安裝Docker
```
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg
sudo mkdir -m 0755 -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker $USER
newgrp docker
```
# 驗證Docker
```
docker --version
```
![alt text](pic\image-7.png)

# Clone Cvat repo
```
git clone -b sst-v24.06.01 https://github.com/smartsurgerytek/dentistry-annotation-cvat.git
cd dentistry-annotation-cvat
```
# build DOCKER
```
docker compose -f docker-compose.yml -f docker-compose.dev.yml build
docker compose -f docker-compose.yml -f components/serverless/docker-compose.serverless.yml up -d
```
![alt text](pic\image-8.png)
安裝的同時，可以開新的posh連進機器去下載Nuclio

# 安裝Nuclio
```
curl -s https://api.github.com/repos/nuclio/nuclio/releases/latest \
			| grep -i "browser_download_url.*nuctl.*$(uname)" \
			| cut -d : -f 2,3 \
			| tr -d \" \
			| wget -O nuctl -qi - && chmod +x nuctl
sudo mv nuctl /usr/local/bin/
```
# 創建Cvat的Project
```
nuctl create project cvat
```
![alt text](pic\image-9.png)
可以使用
```
nuctl get projects
```
來查看創建的project
![alt text](pic\image-10.png)

# Clone smartsurgerytek/dentistry-annotation-nuclio/repo
```
cd ~
git clone https://github.com/smartsurgerytek/dentistry-annotation-nuclio.git
cd dentistry-annotation-nuclio
```
![alt text](pic\image-11.png)
# 把模型部署上去
```
nuctl deploy measurement --path ./src/measurement/ --project-name cvat --platform local
nuctl deploy segmentation --path ./src/segmentation/ --project-name cvat --platform local
```
DOCKER跑完後會出現：
![alt text](pic\image-12.png)
CVAT設定
```
docker exec -it cvat_server python manage.py migrate
docker exec -it cvat_server python manage.py collectstatic
docker exec -it cvat_server python manage.py syncperiodicjobs
docker exec -it cvat_server python manage.py createsuperuser --username admin
```
![alt text](pic\image-13.png)

可以打
```
echo "http://${CVAT_HOST}:8080"
```
![alt text](pic\image-14.png)
![alt text](pic\image-15.png)

# measurement
![alt text](pic\image-16.png)

# segmentation
![alt text](pic\image-17.png)