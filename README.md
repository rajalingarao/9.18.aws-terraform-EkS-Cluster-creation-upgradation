# Infrastructure creation and deletion

```
for i in  10-vpc/ 20-sg/ 30-bastion 40-eks ; do cd $i; terraform init -reconfigure; cd .. ; done 
```

```
for i in  10-vpc/ 20-sg/ 30-bastion 40-eks ; do cd $i; terraform plan; cd .. ; done 
```

```
for i in  10-vpc/ 20-sg/ 30-bastion 40-eks ; do cd $i; terraform apply -auto-approve; cd .. ; done 
```

```
for i in 40-eks/ 30-bastion/ 20-sg/ 10-vpc/ ; do cd $i; terraform destroy -auto-approve; cd .. ; done 
```

# Infrastructure
Creating above infrastructure involves lot of steps, as maintained sequence we need to create
* VPC
* All security groups and rules
* Bastion Host
* EKS

## Sequence
* (Required). create VPC first
* (Required). create SG after VPC
* (Required). create bastion host. It is used to connect RDS and EKS cluster.
* (Optional). VPN, same as bastion but a windows laptop can directly connect to VPN and get access of RDS and EKS.


### Admin activities
**Bastion**
* Connect to Bastion Server:SSH to bastion host
* run below command and configure the credentials.
```
aws configure
```
* Checking connection with AWS:
```
aws s3 ls
```
```
aws sts get-caller-identity
```


* To run a pod in cluster, bastion need to get aws update kubeconfig file
```
aws eks update-kubeconfig --region us-east-1 --name expense-dev
```
```
kubectl get nodes
```
* What is kubeconfig file in Kubernetes?
    *  In eks, kubeconfig is used for authentication and authorization for any cluster kubenetes cluster stored in .kube/config
    * cd .kube
    * cat config
    * This is like .ssh in windows.






```
kubectl apply -f nginx.yaml
```
```
kubectl get pods
```
* Note: pod is not used in EKS upgradation. We can use deployment, if we close pods in one node, it will be pulled in another node. so deployment is the best choice.

Now Cluster is running, few applications are running inside.

# Cluster upgrade is planned

no downtime to the existing apps but no new release and no new deployments in the upgrade time.

For green nodes in the cluster.
terraform plan
terraform apply -auto-approve
```
kubectl get nodes
```
# Cluster upgradation process start here:

1. first we need to get another node group called green.
2. taint/cordon the green nodes, so that they should not get any pods scheduled
    * kubectl taint nodes ip-10-0-11-151.ec2.internal project=expense:NoSchedule (green nodes)
    * kubectl taint nodes ip-10-0-11-142.ec2.internal project=expense:NoSchedule (green nodes)
    OR
    * kubectl cordon ip-10-0-11-111.ec2.internal (green nodes)
    * kubectl cordon ip-10-0-12-90.ec2.internal (green nodes)

3. now upgrade your control plane, do it from AWS console that is 1.29 to 1.30 [Control plan upgradation first]
4. upgrade green node group also 1.29 to 1.30 [Nodes upgradation second]
5. shift the workloads from 1.29 node group to 1.30 means follow below two steps

6. taint/cordon blue nodes.  Note: cordon and uncordon are best options then taint and untaint
    * kubectl taint/cordon ip-10-0-11-219.ec2.internal (blue nodes)
    * kubectl taint/cordon ip-10-0-12-82.ec2.internal (blue nodes)

7. untaint/uncordon green nodes
    * kubectl untaint/uncordon ip-10-0-11-219.ec2.internal
    * kubectl untaint/uncordon ip-10-0-12-82.ec2.internal

8. drain blue nodes
  * USED for pods 
    * kubectl drain --ignore-daemonsets ip-10-0-11-219.ec2.internal
    * kubectl drain --ignore-daemonsets ip-10-0-12-82.ec2.internal  getting error,because pod running
    * kubectl drain --ignore-daemonsets ip-10-0-11-219.ec2.internal --force

  * USED for Deployment:
    * kubectl drain --ignore-daemonsets ip-10-0-11-219.ec2.internal (blue nodes only)
    * kubectl drain --ignore-daemonsets ip-10-0-13-342.ec2.internal (blue nodes only)

```
kubectl get nodes
```
* Note: Please check pods moved from blue nodes to green nodes. Deployment completed.
```
kubectl get pods -n kube-system -o wide
```
* Note: upgrade the blue nodes from 1.29 to 1.30 version and comment the blue nodes in terraform tf file.
* inform all stake holders, application teams. perform sanity testing and close the activity


* Then go to terraform code and comment the blue nodes code and apply it, update 1.29 -->1.30 
    * We will take one hour download to upgrade the platform, not for the applications.
    * inform all stake holders, application teams. perform sanity testing. close the activity

* differences: Pod, Deployment, Daemonset
    * Pod: The basic unit in Kubernetes — runs one or more containers. Not self-healing. Best for testing or temporary tasks.
    * Deployment: Manages multiple copies of a pod. Automatically replaces failed pods and supports zero-downtime updates. Ideal for running scalable apps.
    * DaemonSet: Runs one pod per node, typically for background tasks like logging or monitoring. Used for system-level services, not app scaling.

* Differences on Taint and Cordon:
    * Cordon is best option than taint, if someone applied toleration on nodes, upgradation will not work.
    * Cordon: Stops new pods from being scheduled on a node. Existing pods stay.
    * Taint: Blocks pods unless they tolerate the taint. Can also evict existing pods (with NoExecute).
    * Cordon = temporary block
    * Taint = rule-based filtering

* Note: Once cluster is upgraded, then we upgrade green nodes 
    * rolling update -->if workload running in this nodes, we will do rolling update.
    * force update   -->if no workload running in this nodes, we will do force update.
