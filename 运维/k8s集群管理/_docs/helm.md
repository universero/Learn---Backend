## helm的安装

`sudo snap install helm --classic`

helm需要访问文件和端口，所以需要--classic参数允许访问

## helm添加仓库

`helm repo add bitnami https://charts.bitnami.com/bitnami`
`helm repo update`

根据需要安装的软件调整仓库位置

## 拉取charts包

`hell pull <charts包名>`

`tar -zxvf xxx.tar`解压charts包，以修改默认配置

## 修改value.yml

在解压处理的charts目录中修改对应的配置

## 部署

`helm install <deployment名称> -f <chart路径> --namespace 命名空间 <chart名称>`

## 卸载

`helm uninstall <release的名称> --namespace <命名空间>`

