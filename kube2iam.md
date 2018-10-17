# Kubernetes cluster on AWS


## Prepare:
* Install kubectl: `https://kubernetes.io/docs/tasks/tools/install-kubectl/`
* Install kops: `https://github.com/kubernetes/kops/blob/master/docs/install.md`
* Install AWS CLI (requires python+pip): `https://docs.aws.amazon.com/cli/latest/userguide/installing.html`

* Configure AWS CLI:
  * Have AWS Credentials ready:
    * AWS Access Key ID
    * AWS Secret Access Key
  * Other AWS CLI settings
    * Default Region: us-west-2
    * Default output format: json 
  * Use the following command to configure aws cli: ``aws configure``

## Deploying k8s cluster using kops
Note: cluster-fqdn must be a valid fqdn string


*  Create a new bucket on your S3 and then set env variable: 
    ```/bin/sh
    bucket=kops-aws-state
    aws s3api create-bucket --bucket $bucket --region {{.awsRegion}}
    export KOPS_STATE_STORE=s3://$bucket
    export CLUSTER_NAME=CLUSTER_FQDN
    ```

*  set up DNS
    * create a new route 53 hosted zone, e.g.: `myorg.com`
    * KOPS will create a new record, e.g. `myClusterName.myorg.com` when cluster deployed.

*   We will be using template file `templates/cluster/hacluster.tmpl.yaml` (but it is also possible to create cluster via cli):

    ```yaml
    ---
    apiVersion: kops/v1alpha2
    kind: Cluster
    metadata:
      creationTimestamp: null
      name: {{.clusterName}}
    spec:
      additionalPolicies:
        node: |
          [
            {
              "Effect": "Allow",
              "Action": [
                "route53:ChangeResourceRecordSets"
              ],
              "Resource": [
                "arn:aws:route53:::hostedzone/*"
              ]
            },
            {
              "Effect": "Allow",
              "Action": [
                "route53:ListHostedZones",
                "route53:ListResourceRecordSets"
              ],
              "Resource": [
                "*"
              ]
            },
            {
              "Effect": "Allow",
              "Action": ["sts:AssumeRole"],
              "Resource": ["*"]
            }
          ]
      api:
        loadBalancer:
          type: Public
      authorization:
        rbac: {}
      channel: stable
      cloudProvider: aws
      etcdClusters:
      - etcdMembers:
        - instanceGroup: master-{{.awsRegion}}a
          name: a
        - instanceGroup: master-{{.awsRegion}}b
          name: b
        - instanceGroup: master-{{.awsRegion}}c
          name: c
        name: main
      - etcdMembers:
        - instanceGroup: master-{{.awsRegion}}a
          name: a
        - instanceGroup: master-{{.awsRegion}}b
          name: b
        - instanceGroup: master-{{.awsRegion}}c
          name: c
        name: events
      iam:
        allowContainerRegistry: true
        legacy: false
      kubeAPIServer:
        admissionControl:
        - NamespaceLifecycle
        - LimitRanger
        - ServiceAccount
        - PersistentVolumeLabel
        - DefaultStorageClass
        - DefaultTolerationSeconds
        - MutatingAdmissionWebhook
        - ValidatingAdmissionWebhook
        - ResourceQuota
        - NodeRestriction
        - Priority
        runtimeConfig:
          autoscaling/v2beta1: "true"
      kubeControllerManager:
        horizontalPodAutoscalerUseRestClients: true
      kubelet:
        cloudProvider: aws
        enableCustomMetrics: true
      kubernetesApiAccess:
      - 0.0.0.0/0
      kubernetesVersion: {{.kubernetesVersion}}
      masterInternalName: api.internal.{{.clusterName}}
      masterPublicName: api.{{.clusterName}}
      networkCIDR: 172.20.0.0/16
      networking:
        amazonvpc: {}
      nonMasqueradeCIDR: 172.20.0.0/16
      sshAccess:
      - 0.0.0.0/0
      subnets:
      - cidr: 172.20.32.0/19
        name: {{.awsRegion}}a
        type: Private
        zone: {{.awsRegion}}a
      - cidr: 172.20.64.0/19
        name: {{.awsRegion}}b
        type: Private
        zone: {{.awsRegion}}b
      - cidr: 172.20.96.0/19
        name: {{.awsRegion}}c
        type: Private
        zone: {{.awsRegion}}c
      - cidr: 172.20.0.0/22
        name: utility-{{.awsRegion}}a
        type: Utility
        zone: {{.awsRegion}}a
      - cidr: 172.20.4.0/22
        name: utility-{{.awsRegion}}b
        type: Utility
        zone: {{.awsRegion}}b
      - cidr: 172.20.8.0/22
        name: utility-{{.awsRegion}}c
        type: Utility
        zone: {{.awsRegion}}c
      topology:
        dns:
          type: Public
        masters: private
        nodes: private
      ---

      apiVersion: kops/v1alpha2
      kind: InstanceGroup
      metadata:
        creationTimestamp: null
        labels:
          kops.k8s.io/cluster: {{.clusterName}}
        name: master-{{.awsRegion}}a
      spec:
        image: {{.image.repository}}
        machineType: {{.master.machineType}}
        maxSize: 1
        minSize: 1
        nodeLabels:
          kops.k8s.io/instancegroup: master-{{.awsRegion}}a
        role: Master
        subnets:
        - {{.awsRegion}}a

      ---

      apiVersion: kops/v1alpha2
      kind: InstanceGroup
      metadata:
        creationTimestamp: null
        labels:
          kops.k8s.io/cluster: {{.clusterName}}
        name: master-{{.awsRegion}}b
      spec:
        image: {{.image.repository}}
        machineType: {{.master.machineType}}
        maxSize: 1
        minSize: 1
        nodeLabels:
          kops.k8s.io/instancegroup: master-{{.awsRegion}}b
        role: Master
        subnets:
        - {{.awsRegion}}b

      ---

      apiVersion: kops/v1alpha2
      kind: InstanceGroup
      metadata:
        creationTimestamp: null
        labels:
          kops.k8s.io/cluster: {{.clusterName}}
        name: master-{{.awsRegion}}c
      spec:
        image: {{.image.repository}}
        machineType: {{.master.machineType}}
        maxSize: 1
        minSize: 1
        nodeLabels:
          kops.k8s.io/instancegroup: master-{{.awsRegion}}c
        role: Master
        subnets:
        - {{.awsRegion}}c

      ---

      apiVersion: kops/v1alpha2
      kind: InstanceGroup
      metadata:
        creationTimestamp: null
        labels:
          kops.k8s.io/cluster: {{.clusterName}}
        name: nodes
      spec:
        image: {{.image.repository}}
        machineType: {{.node.machineType}}
        maxSize: {{.node.maxSize}}
        minSize: {{.node.minSize}}
        nodeLabels:
          kops.k8s.io/instancegroup: nodes
        role: Node
        subnets:
        - {{.awsRegion}}a
        - {{.awsRegion}}b
        - {{.awsRegion}}c

      ---

      apiVersion: kops/v1alpha2
      kind: InstanceGroup
      metadata:
        creationTimestamp: null
        labels:
          kops.k8s.io/cluster: {{.clusterName}}
        name: bastions
      spec:
        image: {{.image.repository}}
        machineType: t2.micro
        maxSize: 1
        minSize: 1
        nodeLabels:
          kops.k8s.io/instancegroup: bastions
        role: Bastion
        subnets:
        - utility-{{.awsRegion}}a


    ```
*   Create a values file:  `inventory/myClusterName.company.com.yaml`
    ```yaml
    dnsZone: myClusterName.myorg.com
    awsRegion: us-west-2
    master:
      machineType:  t2.medium
    node:
      machineType: t2.medium
      minSize: 7
      maxSize: 7
    image:
      repository: kope.io/k8s-1.9-debian-jessie-amd64-hvm-ebs-2018-03-11
    kubernetesVersion: 1.10.8

    ```
*  Render kops configuration files:
    ```/bin/bash
    clusterConfigFile
    template_dir=templates/cluster/
    values_file=inventory/myClusterName.company.com.yaml
    mkdir -p output
    clusterConfigFile="output/$CLUSTER_NAME.yaml"
    
    KOPS toolbox template \
        --template ${template_dir} \
        --values ${values_file}  \
        --logtostderr \
        --v 9 \
        --name $CLUSTER_NAME \
        --output $clusterConfigFile
    ```
*  Create a new cluster (without creating AWS resources: `$KOPS create -f ${clusterConfigFile}`
*  Configure ssh `kops create secret --name $CLUSTER_NAME sshpublickey admin -i ~/.ssh/id_rsa.pub`
*  Commit changes:
    ```/bin/bash
    kops update cluster $CLUSTER_NAME  \
        --logtostderr \
        --yes

    while [ 1 ]; do
      echo "# Checking if cluster $CLUSTER_NAME is ready"
      kops validate cluster $CLUSTER_NAME && break || sleep 30
    done;
    ```

## Configuring IAM permissions and using kube2iam
We will be using [kube2iam](https://github.com/jtblin/kube2iam) to assign AWS IAM roles to pods. We will create a DaemonSet and a service account for kube2iam. Kubernetes cluster is deployed into AWS VPC, and it is required to specify networking driver for kube2iam, which is eni+ in our case. 

```yaml
---
# Source: kube2iam/templates/serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    app: kube2iam
  name: kube2iam
---
# Source: kube2iam/templates/clusterrole.yaml
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  labels:
    app: kube2iam
  name: kube2iam
rules:
  - apiGroups:
      - ""
    resources:
      - namespaces
      - pods
    verbs:
      - list
      - watch
---
# Source: kube2iam/templates/clusterrolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  labels:
    app: kube2iam
  name: kube2iam
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kube2iam
subjects:
  - kind: ServiceAccount
    name: kube2iam
    namespace: kube-system
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube2iam
  labels:
    app: kube2iam
spec:
  selector:
    matchLabels:
      name: kube2iam
  template:
    metadata:
      labels:
        name: kube2iam
    spec:
      hostNetwork: true
      serviceAccountName: kube2iam
      containers:
        - image: jtblin/kube2iam:0.10.4
          name: kube2iam
          args:
            - "--auto-discover-base-arn"
            - "--host-interface=eni+"
            - "--host-ip=$(HOST_IP)"
            - "--iptables=true"
          env:
            - name: HOST_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
          securityContext:
            privileged: true
          ports:
            - containerPort: 8181
              hostPort: 8181
              name: http
```

During cluster deployemnt kops created a new AWS IAM Role: `arn:aws:iam::0123456789:role/nodes.CLUSTER_NAME_FQDN`
All nodes in the cluster have that role and permissions (defined in additionalPolicies in the cluster template). Node role allows to assume other roles. Let's set up a new role and trust.

1. create a pod-trust-policy document for our new AWS Role (replace CLUSTER_NAME_FQDN by actual arn of node role). Create a file pod-role-trust-policy.json:
    ```json
    {
        "Version": "2012-10-17",
        "Statement": [
          {
            "Sid": "",
            "Effect": "Allow",
            "Principal": {
              "Service": "ec2.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
          },
          {
            "Sid": "",
            "Effect": "Allow",
            "Principal": {
              "AWS": "arn:aws:iam::0123456789:role/nodes.CLUSTER_NAME_FQDN"
            },
            "Action": "sts:AssumeRole"
          }
        ]
    }
    ```
2. Create a new role:
```/bin/bash
podrole=MyPodRole
aws iam create-role \
  --role-name $podrole \
  --path kube2iam \
  --assume-role-policy-document \
  file://pod-role-trust-policy.json
```  
Find role arn: `arn:aws:iam::0123456789:role/kube2iam/MyPodRole`

3.  Now you are able to assign this role to your pod/deployemnts via annptations:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: aws-cli
  labels:
    name: aws-cli
  annotations:
    iam.amazonaws.com/role: arn:aws:iam::0123456789:role/kube2iam/MyPodRole
spec:
  containers:
    - image: cgswong/aws:aws
      command:
        - "sleep"
        - "9999999"
      name: aws-cli
```


## Testing

  
