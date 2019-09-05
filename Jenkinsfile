def remote = [:]
remote.name = "ubuntu"
remote.allowAnyHosts = true
def ID
def IP
node {
    withCredentials(
        [[
            $class: 'AmazonWebServicesCredentialsBinding',
            accessKeyVariable: 'AWS_ACCESS_KEY_ID',
            credentialsId: 'evaldID',  
            secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
        ]]) {
        stage("create EC2 instance"){
            sh 'aws configure set region us-east-2'
            ID = sh (script: 'aws ec2 run-instances --image-id ami-0f93b5fd8f220e428 --count 1 --instance-type t2.micro --key-name amazonkey --security-group-ids sg-ea228589 --subnet-id subnet-d54bfd99 --region us-east-2 --query \'Instances[0].InstanceId\'',returnStdout: true)
        }
        stage("get the EC2 external ip"){
            remote.host = sh (script: "aws ec2 describe-instances --query \'Reservations[0].Instances[0].PublicIpAddress\' --instance-ids $ID",returnStdout: true)
        }
        stage ("Waiting until initialized"){
            timeout(30) {
                waitUntil {
                  def response = sh(script: "aws ec2 describe-instances --query \'Reservations[0].Instances[0].State.Code\' --instance-ids $ID",returnStatus: true)
                  return (response == 0)
                }
            }
        }
    }
   withCredentials([sshUserPrivateKey(credentialsId: 'aID', keyFileVariable: 'identity', passphraseVariable: '', usernameVariable: 'userName')]) {
      remote.user = userName
      remote.identityFile = identity
      stage("Install Amazon Cli") {
            sh 'sudo apt-get update'
            sh 'sudo apt-get install awscli -y'
      }
      stage("Install kops"){
            sh 'curl -LO https://github.com/kubernetes/kops/releases/download/$(curl -s https://api.github.com/repos/kubernetes/kops/releases/latest | grep tag_name | cut -d \'\"\' -f 4)/kops-linux-amd64 ' 
            sh 'chmod +x kops-linux-amd64'
            sh 'sudo mv kops-linux-amd64 /usr/local/bin/kops'
      }
      stage("Install kubectl"){
            sh 'curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl'
            sh 'chmod +x ./kubectl'
            sh 'sudo mv ./kubectl /usr/local/bin/kubectl'
      }
       withCredentials(
        [[
            $class: 'AmazonWebServicesCredentialsBinding',
            accessKeyVariable: 'AWS_ACCESS_KEY_ID',
            credentialsId: 'cfb29e08-bb19-420f-b13e-079804cbfe52',  
            secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
        ]]) {
            
           stage("Create s3 bucket"){
                sh 'aws configure set region us-east-2'
                sh 'aws s3 mb s3://k8s.evald.in'
            }
            
            stage("Generate ssh-keygen"){
                sh 'sudo chmod -R 700 /root/.ssh/id_rsa'
                sh 'sudo ssh-keygen -t rsa -N "" -f /root/.ssh/id_rsa -y'
            }
            stage("Create cluster configurations"){
                sh 'sudo chmod -R 777 /root/'
                sh 'sudo chmod -R 777 /root/.ssh/'
                sh 'kops create cluster k8s.evald.in --node-count 2 --zones us-east-2b --node-size t2.micro --master-size t2.micro --master-volume-size 8 --node-volume-size 8  --ssh-public-key /root/.ssh/id_rsa.pub --state s3://k8s.evald.in --dns-zone Z229NXDHW3FXAA --dns private --yes'
                sh 'sudo chmod -R 700 /root/.ssh/'
                sh 'sudo chmod -R 700 /root/'
            }
            stage("Create the cluser"){
                sh 'kops update cluster k8s.evald.in --state s3://k8s.evald.in --yes'
            }
            stage('Check Availability') {
                waitUntil{
                    try{        
                        sh "kubectl get nodes"
                        return true
                    }catch (Exception e){
                        return false
                    }
                }
            }
            stage ("Deployment & Replicas"){
                sh 'kubectl run my-app --image=evaldnexhipi/dockerhubi:firsttry --port=8080 --replicas=2' 
            }
            stage ("Exposing the Deployment"){
                sh 'kubectl expose deployment my-app --type=LoadBalancer --port=8080'
            }
            stage ("Autoscaling"){
                sh 'kubectl autoscale deployment my-app --cpu-percent=1 --min=2 --max=5'
            }
            stage ("Updating the image..."){
                //sh 'kubectl set image deployment/my-app my-app=xhesi12/taleas_img:latest'
                //sh 'kubectl rollout undo deployments/my-app'
            }
           stage ("Verifications"){
                sh 'kubectl cluster-info'
                sh 'kubectl get nodes'
                sh 'kubectl version --short'
            }
            
           stage ("Deployment of files"){
                //sh 'rm -r yamlDirectory'
               sh 'git clone https://github.com/XhesikaLeka/yamlDirectory.git'
               sh "sed -i \"s/<<NFS Server IP>>/\"${remote.host}\"/g\" yamlDirectory/deployment.yaml"
            }
            stage ("Deployment of the 3 files"){
                sh 'kubectl create -f yamlDirectory/deployment.yaml' 
                sh 'kubectl create -f yamlDirectory/class.yaml --validate=false'
                sh 'kubectl create -f yamlDirectory/rbac.yaml'
            }
            
            stage ("Helm Installation"){
                sh 'curl -LO https://git.io/get_helm.sh'
                sh 'chmod 700 get_helm.sh'
                sh './get_helm.sh'
            }
        
            stage ("Prometheus pre-Installation"){
               sh 'kubectl -n kube-system create serviceaccount tiller'
                sh 'kubectl create clusterrolebinding tiller --clusterrole cluster-admin --serviceaccount=kube-system:tiller'
                sh 'kubectl get serviceaccounts -n kube-system'
                sh 'helm init --service-account tiller --wait'
            }
            stage ("Waiting function"){
                  sh 'kubectl -n kube-system get pods'
                  sh 'sleep 30'
            }
            stage ("Prometheus Deployment"){
               // sh 'helm del --purge prometheus'
                sh 'helm repo update'
                sh 'helm install stable/prometheus --namespace monitoring --name prometheus --set server.service.type=LoadBalancer --set server.service.servicePort=8082'
            }            
            stage("Defining the grafana data sources"){
                //sh 'helm del --purge grafana'
                sh 'kubectl apply -f https://raw.githubusercontent.com/islajd/kubernetes-prometheus-grafana/master/monitoring/grafana/config.yml'
                sh 'kubectl get configmaps -n monitoring'
            }
            stage("Override Grafana value"){
                sh 'helm install stable/grafana --namespace monitoring --name grafana --set service.type=LoadBalancer --set service.port=8081 --set persistence.enabled=true'
                sh 'kubectl get pods -n monitoring'
            }
            stage("Get the Grafana Password"){
                sh 'kubectl get secret --namespace monitoring grafana -o jsonpath="{.data.admin-password}"| base64 --decode ; echo'
            }
            stage ("Testing"){
                sh 'kubectl describe service prometheus -n monitoring'
                sh 'kubectl describe service grafana -n monitoring'
            }
        }
    }
}
