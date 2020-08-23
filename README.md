# Wordpress kubernetes S3
This tutorial shows you hot to deploy a Wordpress site use external Mysql DataBase, PersistentVolumes and PersistentVolumeClaims to store data.

**References**
- [Deploying WordPress and MySQL with Persistent Volumes](https://kubernetes.io/docs/tutorials/stateful-application/mysql-wordpress-persistent-volume/)
- [Use Amazon S3 with Wordpress to securely deliver media files](https://aws.amazon.com/pt/blogs/compute/deploying-a-highly-available-wordpress-site-on-amazon-lightsail-part-2-using-amazon-s3-with-wordpress-to-securely-deliver-media-files/)

### Create cluster
We will use the eksctl to deploy our cluster.
> Ensure to replace <CLUSTER_NAME> with your clustername
```
eksctl create cluster -f cluster.yaml
```

### Provision Mysql DB
Provision an Aurora MySQL Database or your preferred Mysql DB instance
> Ensure, you create a security with 3306 open to the cluster nodes (this can be done via a CIDR range or the Security Group created for the node group). 

##### Create DB Secret
```
kubectl create secret generic wordpress-db --from-literal=username=admin --from-literal=name=wp-natura-db --from-literal=host='wp-natura-db.c9oh0n6cx98q.sa-east-1.rds.amazonaws.com' --from-literal=password='deploy102030'
```

### Provision EFS File System
Then navigate to the EFS console, and create a mount point in the same VPC. Again, ensure you create a Security Group that allows NFS traffic from the node group.

We need to deploy the EFS CSI Driver so that our containers can mount our EFS share. To deploy the driver run the following command:
```
kubectl apply -k "github.com/kubernetes-sigs/aws-efs-csi-driver/deploy/kubernetes/overlays/stable/?ref=master"
```

#### To create an Amazon EFS file system for your Amazon EKS cluster
[AWS Documentation](https://docs.aws.amazon.com/eks/latest/userguide/efs-csi.html)

1. Locate the VPC ID for your Amazon EKS cluster. You can find this ID in the Amazon EKS console, or you can use the following AWS CLI command. 
```
aws eks describe-cluster --name cluster_name --query "cluster.resourcesVpcConfig.vpcId" --output text
Output:
vpc-04ddc34c94f5356b3
```

2. Locate the CIDR range for your cluster's VPC. You can find this in the Amazon VPC console, or you can use the following AWS CLI command. 
```
aws ec2 describe-vpcs --vpc-ids vpc-04ddc34c94f5356b3 --query "Vpcs[].CidrBlock" --output text
Output:
10.1.0.0/16
```

3. Create the Amazon EFS file system for your Amazon EKS cluster.

4. Configure a security group that allows inbound NFS traffic for your Amazon EFS mount points.
 
### Create Salt Secret
[Generate Salt keys](https://api.wordpress.org/secret-key/1.1/salt/)
Use WordPress web generator to create salt keys and replace in the command below

```
kubectl create secret generic wordpress-salts --from-literal=auth_key=<AUTH_KEY> \
    --from-literal=secure_auth_key=<SECURE_AUTH_KEY> \
    --from-literal=logged_in_key=<LOGGED_IN_KEY> \
    --from-literal=nonce_key=<NONCE_KEY> \
    --from-literal=auth_salt=<AUTH_SALT> \
    --from-literal=secure_auth_salt=<SECURE_AUTH_SALT> \
    --from-literal=logged_in_salt=<LOGGED_IN_SALT> \
    --from-literal=nonce_salt=<NONCE_SALT>

```

### Create Ingress Controller
Now we will deploy the ALB Ingress Controller. 
Make sure you replace <YOUR VPC ID> and <CLUSTER NAME> specified in the cluster.yml.
```
kubectl apply -f alb-ingress-controller.yaml
```

### Create S3 Bucket
[See how to](https://aws.amazon.com/pt/blogs/compute/deploying-a-highly-available-wordpress-site-on-amazon-lightsail-part-2-using-amazon-s3-with-wordpress-to-securely-deliver-media-files/)


###### Create S3 User Secret
> Ensure you replace <ACCESS_KEY> and <SECRET_KEY> with the AWS user keys.
```
kubectl create secret generic s3-user --from-literal=access_key=<ACCESS_KEY> --from-literal=secret_key=<SECRET_KEY>
```


### Deploy Wordpress
> Ensure you replace <FILE SYSTEM ID> with the ID of your EFS share and update the database configuration with the credentials you used to create the database.
```
kubectl apply -f wordpress.yaml
```

### Deploy Ingress Endpoint
It's a simple ingress example, that generate a url endpoint like `http://77869463-default-wordpress-9484-1528138773.sa-east-1.elb.amazonaws.com/` just to be easy.
It's not the intention to explore how to configure load balance and external access controller
```
kubectl apply -f ingress.yaml
kubectl get ingress
```

### Delete Cluster
```
eksctl delete cluster --region=sa-east-1 --name=<CLUSTER_NAME>
```
