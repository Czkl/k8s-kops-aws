操作系统 MAC OS
一、准备环境
    1、安装kubectl
    2、安装kops
    brew install kubectl kops
    注意：安装后如果命令行不能使用kops或kubectl命令，那么需要手动设置软链接

    3、安装AWS CLI 工具
    https://docs.aws.amazon.com/zh_cn/cli/latest/userguide/install-macos.html  （官网安装文档）

    4、安装kops-cn （aws中国区专用，直接用kops安装会比较麻烦且实例运行状态有问题）
    参考 https://github.com/nwcdlabs/kops-cn 或 https://swift.ctolib.com/nwcdlabs-kops-cn.html
    下载 https://github.com/nwcdlabs/kops-cn/archive/master.zip
    解压到 /Users/haohan.peng/beach/dolphin/kops-cn-master

    5、密钥文件生成到目录/Users/haohan.peng/beach/dolphin/kops-cn-master
    cd /Users/haohan.peng/beach/dolphin/kops-cn-master
    ssh-keygen -t rsa -C dolphin-k8s -f dolphin-k8s.pem
    公钥文件 dolphin-k8s.pem.pub
    私钥文件 dolphin-k8s.pem

二、kops账号准备
    https://blog.csdn.net/qq_36348557/article/details/79795288
    1、配置已有的aws账户 dolphin
    aws configure
    依次按回车配置下面4项：
    AWS Access Key ID
    AWS Secret Access Key
    Default region name 设为 cn-northwest-1 对应中国(宁夏)
    Default output format 设为 json

    2、配置变量
    export AWS_ACCESS_KEY_ID=$(aws configure get aws_access_key_id)
    export AWS_SECRET_ACCESS_KEY=$(aws configure get aws_secret_access_key)
    export AWS_REGION=$(aws configure get region)
    export TARGET_REGION=cn-northwest-1
    export AWS_PROFILE=default
    export KOPS_S3_DOMAIN=dolphin-test-state-store
    export KOPS_STATE_STORE=s3://$KOPS_S3_DOMAIN
    export VPCID=vpc-06b8c09d76f1fb9d3
    export MASTER_COUNT=3
    export MASTER_SIZE=t2.medium
    export NODE_SIZE=t2.medium
    export NODE_COUNT=2
    export KOPS_CN_PATH=~/exercise/Kops-aws-k8s/kops-cn-master
    export SSH_PUBLIC_KEY=$KOPS_CN_PATH/k8s-test.pem.pub
    export KUBERNETES_VERSION=v1.15.5
    export KOPS_VERSION=1.15.0
    export CUSTOM_CLUSTER_NAME=test-pe.k8s.local
    可执行
    cd /Users/haohan.peng/beach/dolphin/kops-cn-master
    source ./export-env.sh

    3、创建kops用户组，并将自己的账号加入该组
    aws iam create-group --group-name kops
    aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess --group-name kops
    aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonRoute53FullAccess --group-name kops
    aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess --group-name kops
    aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/IAMFullAccess --group-name kops
    aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonVPCFullAccess --group-name kops
    aws iam add-user-to-group --user-name dolphin --group-name kops

    4、创建S3桶
    aws s3api create-bucket \
    --bucket $KOPS_S3_DOMAIN \
    --region $AWS_REGION \
    --create-bucket-configuration LocationConstraint=$AWS_REGION

    aws s3api put-bucket-versioning \
    --bucket $KOPS_S3_DOMAIN \
    --versioning-configuration Status=Enabled
    可执行 ./kops-pre.sh

三、开始执行kops-cn
    1、make create-cluster
    2、make edit-cluster
    拷贝spec.yml 中内容贴到spec节点下，然后保存
    3、make update-cluster
    4、如果需要删除
    make delete-cluster

四、解决AWS中国区坑点
    修改ELB对应的security group，允许8443。
    修改ELB的Listener，对应8443到instance的443。
    修改~/.kube/config文件中的Server的url，后面加上端口号(:8443)。

    kubectl get nodes成功。

    1、在负载均衡中把监听器端口的入口改为28443，并在安全组中放行端口28443
    2、修改本地kops的server端口为28443
    export K8S_ENDPOINT=`kubectl config view -o jsonpath="{.clusters[0].cluster.server}"`
    kubectl config set-cluster dolphin.k8s.local --server=$K8S_ENDPOINT:28443
    或者手动修改 vim ~/.kube/config
    3、检查状态
    kops validate cluster

五、安装K8S Dashboard
    1、ssh连接方式到master
    ssh -i "$KOPS_CN_PATH/dolphin-k8s.pem" admin@ec2-52-82-59-169.cn-northwest-1.compute.amazonaws.com.cn

    2、安装Dashboard并查看安装结果
    kubectl apply -f https://raw.githubusercontent.com/Czkl/k8s-kops-aws/master/kubernetes-dashboard.yaml
    kubectl --namespace=kube-system get deployment kubernetes-dashboard
    kubectl --namespace=kube-system get service kubernetes-dashboard

    删除Dashboard
    kubectl delete -f https://raw.githubusercontent.com/Czkl/k8s-kops-aws/master/kubernetes-dashboard.yaml

    3、修改master的安全组，放开端口30443。
    4、访问Dashboard页面
    https://ec2-52-82-59-169.cn-northwest-1.compute.amazonaws.com.cn:30443/#!/login
    5、如果希望本地访问Dashboard页面
        通过 kubectl porxy
        然后访问
        http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
    服务端端口转发
    kubectl port-forward svc/kubernetes-dashboard -n kube-system 28443:443
    本地端口转发
    ssh -i "$KOPS_CN_PATH/dolphin-k8s.pem" admin@ec2-52-82-59-169.cn-northwest-1.compute.amazonaws.com.cn -L 28443:127.0.0.1:28443

    5、解决认证问题
    直接允许跳过登录
    kubectl apply -f https://raw.githubusercontent.com/Czkl/k8s-kops-aws/master/dashboard-adminuser.yaml
    kubectl apply -f https://raw.githubusercontent.com/TW-China/dolphin_infra/master/dashboard-admin.yml
    或者获取token后登录
    kubectl -n kube-system describe $(kubectl -n kube-system get secret -n kube-system -o name | grep namespace) | grep token

六、手动部署网站（可在jenkins完成）
    1、登录到master执行部署脚本(也可在本地部署)
    kubectl create -f https://raw.githubusercontent.com/TW-China/dolphin-frontend/master/jenkins/scripts/deploy_aws.yaml

    2、查看部署结果
    kubectl get pods -o wide

    3、新建负载均衡器，指向部署的应用(port公网端口，target-port应用端口)，国内访问域名生效需要几分钟
    kubectl expose deployment dolphin-front --type=LoadBalancer --port=28080 --target-port=80
    查看效果
    kubectl get services
