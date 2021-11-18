# Traefik + HTTP3

## Prerequesites

- [eksctl](https://docs.aws.amazon.com/eks/latest/userguide/create-cluster.html)

## Getting started

- Create kubernetes cluster with [eksctl](https://docs.aws.amazon.com/eks/latest/userguide/create-cluster.html)

```bash
export AWS_REGION=eu-north-1
export CLUSTER_NAME=traefik-http3
export KUBECONFIG=${HOME}/.kube/traefik-http3

## Create the EKS cluster
eksctl create cluster \
 --name ${CLUSTER_NAME} \
 --region ${AWS_REGION} \
 --version 1.21
```

> It takes arround 20 minutes to have the cluster up and running.

- Check that you can access to the created cluster

```bash
kubectl get nodes

NAME                                            STATUS   ROLES    AGE    VERSION
ip-192-168-45-130.eu-north-1.compute.internal   Ready    <none>   102s   v1.21.5-eks-bc4871b
ip-192-168-87-177.eu-north-1.compute.internal   Ready    <none>   104s   v1.21.5-eks-bc4871b
```

- Install traefik

Here we will install Traefik Ingress Controller into our EKS cluster to support HTTP/3.

This is a classic Traefik installation in kubernetes, we will first install the CRDs, and creates some RBAC to allow traefik to communication with the Kubernetes API and then create a deployment.

The important part is on the [deployment](./traefik/2-deployment.yaml), because we have to configure Traefik to support HTTP/3.

This feature is experimental on Traefik, so we will have to add the following argument: `--experimental.http3` in the static configuration.

Once it is activated, we have to enable HTTP/3 on the websecure entrypoint with this argument: `--entrypoints.websecure.http3`

- Install Traefik

```bash
kubectl apply -f traefik/
```

- Check that traefik is up and running.

```bash
kubectl get pods
```

One he is running we have to find the way to expose it on the internet.

Let's create a Kubernetes service of Type LoadBalancer and add annotations to ask for a NetworkLoadBalancer on AWS.
HTTP/3 is running on QUIC, so it means that we need to have a LoadBalancer listening on TCP and UDP on the same port.

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: traefik
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb" # Ask for a Network Load Balancer on AWS
spec:
  type: LoadBalancer
  ports:
    - protocol: TCP
      name: web
      port: 80
    - protocol: TCP # We have to listen on TCP on port 443
      name: websecure-tcp
      port: 443
    - protocol: UDP # We have to listen on UDP on port 443 for HTTP/3
      name: websecure-udp
      port: 443
  selector:
    app: traefik
```

- Deploy the LoadBalancer service

```bash
kubectl apply -f traefik/services/3-service-lb.yaml

The Service "traefik" is invalid: spec.ports: Invalid value: []core.ServicePort{core.ServicePort{Name:"web", Protocol:"TCP", AppProtocol:(*string)(nil), Port:80, TargetPort:intstr.IntOrString{Type:0, IntVal:80, StrVal:""}, NodePort:0}, core.ServicePort{Name:"websecure-tcp", Protocol:"TCP", AppProtocol:(*string)(nil), Port:443, TargetPort:intstr.IntOrString{Type:0, IntVal:443, StrVal:""}, NodePort:0}, core.ServicePort{Name:"websecure-udp", Protocol:"UDP", AppProtocol:(*string)(nil), Port:443, TargetPort:intstr.IntOrString{Type:0, IntVal:443, StrVal:""}, NodePort:0}}: may not contain more than 1 protocol when type is 'LoadBalancer'
```

> As you can see in the ouput, we are not authorized to create a LoadBalancer listing on both TCP and UDP at the same time. It is due to a limitation on the [AWS LoadBalancer Controller](https://github.com/kubernetes-sigs/aws-load-balancer-controller/issues/1608#issuecomment-937346660)


As a workarround, we will have to create a service NodePort and creates the NLB manually.

- Deploy the NodePort service

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: traefik
spec:
  type: NodePort
  ports:
    - protocol: TCP
      name: web
      targetPort: 80
      nodePort: 30442
      port: 80
    - protocol: TCP
      name: websecure-tcp
      targetPort: 443
      nodePort: 30443
      port: 443
    - protocol: UDP
      name: websecure-udp
      targetPort: 443
      nodePort: 30443
      port: 443
  selector:
    app: traefik
```

```bash
## Deploy a NodePort service.
kubectl apply -f traefik/services/3-service-node-port.yaml
```

- Get useful info about the clutser:

```bash
## Retrieve the VPC ID.
VPC_ID=$(aws eks describe-cluster --name ${CLUSTER_NAME} --query 'cluster.resourcesVpcConfig.vpcId' --output=text)

## Retrieve the node group ID.
NODE_GROUP_ID=$(aws eks list-nodegroups --cluster-name=${CLUSTER_NAME} --query 'nodegroups[0]' --output text)

## Retrieve instances subnet IDs.
INSTANCES_SUBNET_IDS=$(aws ec2 describe-instances --filters Name=network-interface.vpc-id,Values=$VPC_ID "Name=tag:eks:nodegroup-name,Values=$NODE_GROUP_ID" --query 'Reservations[*].Instances[*].SubnetId' --output text | tr '\n' ' ')
```

- Create the NLB

```bash
## Create the Network Load Balancer with subnets retrieve just before.
aws elbv2 create-load-balancer --name ${CLUSTER_NAME} \
  --type network \
  --subnets $(echo ${INSTANCES_SUBNET_IDS})
```

- Create Target group for the NLB
```bash
TG_NAME=${CLUSTER_NAME}-tg

## Create a TCP target group for web entrypoint on port 30442.
aws elbv2 create-target-group --name ${TG_NAME}-web --protocol TCP --port 30442 --vpc-id ${VPC_ID} \
  --health-check-protocol TCP \
  --health-check-port 30442 \
  --target-type instance

## Create a TCP_UDP target group for websecure entrypoint on port 30443.
aws elbv2 create-target-group --name ${TG_NAME}-websecure --protocol TCP_UDP --port 30443 --vpc-id ${VPC_ID} \
  --health-check-protocol TCP \
  --health-check-port 30443 \
  --target-type instance
```

- Register Instances in the target group

```bash
## Retrieve instances IDs.
INSTANCES=$(kubectl get nodes -o json | jq -r ".items[].spec.providerID"  | cut -d'/' -f5)
IDS=$(for x in `echo ${INSTANCES}`; do echo Id=$x ; done | tr '\n' ' ')
echo $IDS

## Retrieve the target group Arn for the web entrypoint.
TG_ARN_WEB=$(aws elbv2 describe-target-groups --query 'TargetGroups[?TargetGroupName==`traefik-http3-tg-web`].TargetGroupArn' --output text)

## Register instances to the previous TargetGroup web.
aws elbv2 register-targets --target-group-arn ${TG_ARN_WEB} --targets $(echo ${IDS})

## Retrieve the target group Arn for the websecure entrypoint.
TG_ARN_WEBSECURE=$(aws elbv2 describe-target-groups --query 'TargetGroups[?TargetGroupName==`traefik-http3-tg-websecure`].TargetGroupArn' --output text)
## Register instances to the previous TargetGroup websecure.
aws elbv2 register-targets --target-group-arn ${TG_ARN_WEBSECURE} --targets $(echo ${IDS})
```

- Create listener

```bash

## Retrieve the NLB Arn
LB_ARN=$(aws elbv2 describe-load-balancers --names ${CLUSTER_NAME} --query 'LoadBalancers[0].LoadBalancerArn' --output text)

## Create a TCP listener on port 80 that will forward to the TargetGroup traefik-http3-web
aws elbv2 create-listener --load-balancer-arn ${LB_ARN} \
  --protocol TCP --port 80 \
  --default-actions Type=forward,TargetGroupArn=${TG_ARN_WEB}

## Create a TCP_UDP listener on port 443 that will forward to the TargetGroup traefik-http3-websecure
aws elbv2 create-listener --load-balancer-arn ${LB_ARN} \
  --protocol TCP_UDP --port 443 \
  --default-actions Type=forward,TargetGroupArn=${TG_ARN_WEBSECURE}
```

- Configure Instances security groups

```bash
# Retrieve the security groups for each instances.
SGs=$(for x in $(echo $INSTANCES); do aws ec2 describe-instances --filters Name=instance-id,Values=$x \
 --query 'Reservations[*].Instances[*].SecurityGroups[0].GroupId' --output text ; done | sort | uniq)

for x in $(echo $SGs); do
  echo SG=$x;
  aws ec2 authorize-security-group-ingress --group-id $x --protocol tcp --port 30442 --cidr 0.0.0.0/0;
  aws ec2 authorize-security-group-ingress --group-id $x --protocol tcp --port 30443 --cidr 0.0.0.0/0;
  aws ec2 authorize-security-group-ingress --group-id $x --protocol udp --port 30443 --cidr 0.0.0.0/0;
done

## Retrieve the NLB Arn and extract the ID.
NLB_NAME_ID=$(aws elbv2 describe-load-balancers --names ${CLUSTER_NAME} --query 'LoadBalancers[0].LoadBalancerArn' --output text | awk -F":loadbalancer/" '{print $2}')

## Retrieve the NLS DNS name.
NLB_DNS_NAME=$(aws elbv2 describe-load-balancers --names ${CLUSTER_NAME} --query 'LoadBalancers[0].DNSName' --output text)
```


## Deploy

- Deploy the dashboard IngressRoute

```bash
kubectl apply -f traefik/dashboard
```

- Open the dashboard

```bash
open https://${NLB_DNS_NAME}/dashboard/
```

- Let's deploy a demo application

```bash
kubectl apply -f app/
```

- Access to the application

```bash
open https://${NLB_DNS_NAME}/
```

- Check with http3check

```bash
open https://http3check.net/?host=${NLB_DNS_NAME}
```


# Cleanup

```bash
## Delete the Network Load Balancer.
aws elbv2 delete-load-balancer --load-balancer-arn ${LB_ARN}

## Delete the 2 target groups created.
aws elbv2 delete-target-group --target-group-arn ${TG_ARN_WEB}
aws elbv2 delete-target-group --target-group-arn ${TG_ARN_WEBSECURE}

## Delete the EKS cluster.
eksctl delete cluster --name ${CLUSTER_NAME} --region ${AWS_REGION}
```
