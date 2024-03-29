
pipeline {
    agent {
                docker {
                    image 'ruby:2.6'
                    args '-u root:root -v $HOME/workspace/EKS:/EKS'
                    

                }
    }

    environment {
        AWS_ACCESS_KEY_ID  = credentials('AWS_ACCESS_KEY_IDII')
        AWS_SECRET_ACCESS_KEY = credentials('AWS_SECRET_ACCESS_KEYII')
        AWS_DEFAULT_REGION = credentials('AWS_DEFAULT_REGION')
        EKS_CLUSTER_NAME = credentials('EKS_CLUSTER_NAME')
        AWS_ACCOUNT_ID = credentials('AWS_ACCOUNT_IDI')
    }

    stages {
        stage('Provision Remote Backend Move State') {
            steps {
                sh 'apt-get update && apt-get install -y gnupg software-properties-common curl python3'
                sh 'curl -fsSL https://apt.releases.hashicorp.com/gpg | apt-key add - '
                sh 'apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"'
                sh 'apt-get update && apt-get install terraform'
                sh 'terraform init'
                sh 'terraform destroy -auto-approve'
                sh 'python3 append.py'
                sh 'terraform init -force-copy'
            }
        }

        stage('Provision') {
            steps {
                sh 'apt-get update && apt-get install -y gnupg software-properties-common curl'
                sh 'curl -fsSL https://apt.releases.hashicorp.com/gpg | apt-key add - '
                sh 'apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"'
                sh 'apt-get update && apt-get install terraform'
                dir('kubernetes') {
                    sh 'terraform init'
                    sh 'terraform destroy -auto-approve'
                }
            }
        }

        stage('IngressRole') {
            steps {
                    sh 'apt-get update'
                    sh 'apt-get install -y awscli'
                    sh 'curl --silent --location "https://github.com/weaveworks/eksctl/releases/download/v0.70.0/eksctl_Linux_amd64.tar.gz" | tar xz -C /tmp'
                    sh 'mv /tmp/eksctl /usr/local/bin'
                dir('kubernetes') {
                    sh 'aws iam create-policy --policy-name ALBIngressControllerIAMPolicy --policy-document file://iam_policy.json'
                    sh "eksctl utils associate-iam-oidc-provider --region=$AWS_DEFAULT_REGION --cluster=$EKS_CLUSTER_NAME --approve"
                    sh "eksctl create iamserviceaccount \
                        --region=$AWS_DEFAULT_REGION \
                        --cluster=$EKS_CLUSTER_NAME \
                        --namespace=kube-system \
                        --name=aws-load-balancer-controller \
                        --attach-policy-arn=arn:aws:iam::$AWS_ACCOUNT_ID:policy/ALBIngressControllerIAMPolicy \
                        --approve \
                        --override-existing-serviceaccounts"
                }
            }
        }

        stage('AWSIngress') {
            steps {
                //sh 'apt-get update'
                sh ' apt-get install -y apt-transport-https ca-certificates curl awscli'
                sh 'curl https://baltocdn.com/helm/signing.asc |  apt-key add - '
                sh 'echo "deb https://baltocdn.com/helm/stable/debian/ all main" |  tee /etc/apt/sources.list.d/helm-stable-debian.list'
                //sh 'apt-get update'
                sh 'curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg'
                sh 'echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | tee /etc/apt/sources.list.d/kubernetes.list'
                sh 'apt-get update'
                sh 'apt-get install helm'
                sh 'apt-get install -y kubectl'
                sh 'aws eks  update-kubeconfig --name $EKS_CLUSTER_NAME --region $AWS_DEFAULT_REGION'
                sh 'kubectl apply -k "github.com/aws/eks-charts/stable/aws-load-balancer-controller//crds?ref=master"'
                sh 'helm repo add eks https://aws.github.io/eks-charts'
                sh 'helm repo update '
                sh "helm upgrade -i aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=$EKS_CLUSTER_NAME --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller"
            }
        }

        stage('Deploy') {
            steps {
                sh 'apt-get update'
                sh 'apt-get install -y apt-transport-https ca-certificates curl awscli'
                sh 'curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg'
                sh 'echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | tee /etc/apt/sources.list.d/kubernetes.list'
                sh 'apt-get update'
                sh 'apt-get install -y kubectl'
                sh 'aws eks update-kubeconfig --name $EKS_CLUSTER_NAME --region $AWS_DEFAULT_REGION'

                dir('kubernetes') {
                    sh 'kubectl apply -f namespace.yaml'
                    sh 'kubectl apply -f service.yaml'
                    sh 'kubectl apply -f ingress.yaml'
                    sh 'kubectl apply -f deployment.yaml'
                }
            }
        }
    }

// //   post {
// //       always {
// //       cleanWs()
// //       }
// //   }
}
