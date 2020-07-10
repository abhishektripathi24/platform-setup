# Kubernetes [Bare-metal setup with single control-panel]
<img src="https://github.com/abhishektripathi24/platform-setup/blob/master/kubernetes/images/kubernetes-logo.png" width="320" height="200"/>

To know more about Kubernetes, visit https://kubernetes.io/docs/home/

## Overview
From the official docs -

> Kubernetes is an open source container orchestration engine for automating deployment, scaling, and management of containerized applications. The open source project is hosted by the Cloud Native Computing Foundation (CNCF).

## Setup
Installation of `Kubernetes 1.17.2` on `Ubuntu 18.04.3 LTS` using `kubeadm` - [ref](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)

1. Install docker
    ```bash
    sudo apt-get update && sudo apt-get install -y \
        apt-transport-https \
        ca-certificates \
        curl \
        gnupg-agent \
        software-properties-common
    
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
    
    sudo apt-key fingerprint 0EBFCD88
    
    sudo add-apt-repository \
       "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
       $(lsb_release -cs) \
       stable"
    
    ## Install Docker CE.
    sudo apt-get update && sudo apt-get install -y docker-ce docker-ce-cli containerd.io
 
    # Setup daemon.
    sudo vim /etc/docker/daemon.json
    {
      "exec-opts": ["native.cgroupdriver=systemd"],
      "log-driver": "json-file",
      "log-opts": {
        "max-size": "100m"
      },
      "storage-driver": "overlay2"
    }

    # Reload and restart docker daemon 
    sudo systemctl daemon-reload  && sudo systemctl restart docker
 
    sudo docker --version
 
    sudo docker run hello-world
    ```

2. Install kubernetes components
    * Prerequisites -
        ```bash
        # Disable swap on all machines
        sudo swapoff -a # Temporary
        comment entry with type swap in `/etc/fstab` # Permanently
  
        # Configure hostname on master/control-panel node
        sudo hostnamectl set-hostname kubernetes-master-node
  
        # Map etcd data directory to an externally mounted volume on control-panel node
        sudo ln -s /data/etcd /var/lib/etcd
    
        # Configure hostname on each worker node (e.g. for worker 1)  
        sudo hostnamectl set-hostname kubernetes-worker-node-1
        ```
    * Install kubeadm, kubelet, kubectl on all nodes
        ```bash
        sudo apt-get update && sudo apt-get install -y apt-transport-https curl
  
        curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
  
        echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
  
        sudo apt-get update 
  
        # List available versions
        curl -s https://packages.cloud.google.com/apt/dists/kubernetes-xenial/main/binary-amd64/Packages | grep Version | awk '{print $2}'

        # Install from the available versions
        sudo apt-get install -y kubelet=1.17.2-00 kubeadm=1.17.2-00 kubectl=1.17.2-00
        
        # Or install the latest version
        sudo apt-get install -y kubelet kubeadm kubectl
        ```
    * Initializing your control-plane node
        ```bash
        # Initializing your control-plane node
        sudo kubeadm init --pod-network-cidr=192.168.0.0/16
    
        # Once you Kubernetes control-plane has initialized successfully!
        # To start using your cluster, you need to run the following as a regular user:
        mkdir -p $HOME/.kube
        sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
        sudo chown $(id -u):$(id -g) $HOME/.kube/config

        # Installing a Pod network add-on so that your Pods can communicate with each other. By default, Calico uses 192.168.0.0/16 as the Pod network CIDR, though this can be configured in the calico.yaml file. For Calico to work correctly, you need to pass this same CIDR to the kubeadm init command using the --pod-network-cidr=192.168.0.0/16 flag or via the kubeadm configuration.
        kubectl apply -f https://docs.projectcalico.org/v3.11/manifests/calico.yaml
        
        # Once a Pod network has been installed, you can confirm that it is working by checking that the CoreDNS Pod is Running in the output of
        kubectl get pods --all-namespaces
  
        ---
        # Sample outputs after following the process so far:
   
        ubuntu@k8s-master:~$ kubectl get pods --all-namespaces
        NAMESPACE     NAME                                          READY   STATUS    RESTARTS   AGE
        kube-system   calico-kube-controllers-75d56dfc47-j4bcw      1/1     Running   0          2m9s
        kube-system   calico-node-dk9t9                             1/1     Running   0          2m10s
        kube-system   coredns-66bff467f8-cr792                      1/1     Running   0          4m6s
        kube-system   coredns-66bff467f8-x6dmj                      1/1     Running   0          4m6s
        kube-system   etcd-k8s-master                               1/1     Running   0          4m15s
        kube-system   kube-apiserver-k8s-master                     1/1     Running   0          4m15s
        kube-system   kube-controller-manager-k8s-master            1/1     Running   0          4m15s
        kube-system   kube-proxy-98jjz                              1/1     Running   0          4m6s
        kube-system   kube-scheduler-k8s-master                     1/1     Running   0          4m15s
        
        ubuntu@k8s-master:~$ kubectl get namespaces -o wide
        NAME              STATUS   AGE
        default           Active   48m
        kube-node-lease   Active   48m
        kube-public       Active   48m
        kube-system       Active   48m
        ```
    * Once the CoreDNS Pod is up and running, you can continue by joining your worker nodes
        ```bash
        sudo kubeadm join --token <token> <control-plane-host>:<control-plane-port> --discovery-token-ca-cert-hash sha256:<hash>
        # e.g. kubeadm join 10.8.73.199:6443 --token wl5s1e.l58ac9jy6jwqiq8h \
                   --discovery-token-ca-cert-hash sha256:c8c991cfe7ad758334caadf6e9624528ac92303ab7d8a6cd65986c6549cf9699
        
        # If you do not have the token, you can get it by running the following command on the control-plane node:
        kubeadm token list
  
        # By default, tokens expire after 24 hours. If you are joining a node to the cluster after the current token has expired, you can create a new token by running the following commands on the control-plane node:
        kubeadm token create
        openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | \
           openssl dgst -sha256 -hex | sed 's/^.* //'

        # Verify all the joined nodes by running following on control-panel node
        kubectl get nodes -o wide
        
        ---
        # Sample output after joining 2 worker nodes
  
        ubuntu@k8s-master:~$ kubectl get pods --all-namespaces -o wide --watch
        NAMESPACE     NAME                                          READY   STATUS    RESTARTS   AGE   IP              NODE           NOMINATED NODE   READINESS GATES
        kube-system   calico-kube-controllers-75d56dfc47-j4bcw      1/1     Running   0          31m   192.168.56.66   k8s-master     <none>           <none>
        kube-system   calico-node-dk9t9                             1/1     Running   0          31m   10.11.16.59     k8s-master     <none>           <none>
        kube-system   calico-node-h96l5                             1/1     Running   0          17m   10.11.16.63     k8s-worker-2   <none>           <none>
        kube-system   calico-node-pmbcp                             1/1     Running   0          21m   10.11.16.61     k8s-worker-1   <none>           <none>
        kube-system   coredns-66bff467f8-cr792                      1/1     Running   0          33m   192.168.56.67   k8s-master     <none>           <none>
        kube-system   coredns-66bff467f8-x6dmj                      1/1     Running   0          33m   192.168.56.65   k8s-master     <none>           <none>
        kube-system   etcd-k8s-master                               1/1     Running   0          33m   10.11.16.59     k8s-master     <none>           <none>
        kube-system   kube-apiserver-k8s-master                     1/1     Running   0          33m   10.11.16.59     k8s-master     <none>           <none>
        kube-system   kube-controller-manager-k8s-master            1/1     Running   0          33m   10.11.16.59     k8s-master     <none>           <none>
        kube-system   kube-proxy-98jjz                              1/1     Running   0          33m   10.11.16.59     k8s-master     <none>           <none>
        kube-system   kube-proxy-f2nfl                              1/1     Running   0          17m   10.11.16.63     k8s-worker-2   <none>           <none>
        kube-system   kube-proxy-tbbhc                              1/1     Running   0          21m   10.11.16.61     k8s-worker-1   <none>           <none>
        kube-system   kube-scheduler-k8s-master                     1/1     Running   0          33m   10.11.16.59     k8s-master     <none>           <none>
  
        # The following containers should be running at all the worker nodes
        ubuntu@k8s-worker-2:~$ sudo docker ps
        CONTAINER ID        IMAGE                   COMMAND                  CREATED             STATUS              PORTS               NAMES
        449d7bb47594        calico/node             "start_runit"            33 minutes ago      Up 33 minutes                           k8s_calico-node_calico-node-h96l5_kube-system_74fa2a06-0378-4324-bd72-8332bdc5d4df_0
        710d21c15869        k8s.gcr.io/kube-proxy   "/usr/local/bin/kube…"   35 minutes ago      Up 35 minutes                           k8s_kube-proxy_kube-proxy-f2nfl_kube-system_98578f18-1a0d-4056-8fc5-fc4bd48ac1d1_0
        d579a9fbbe95        k8s.gcr.io/pause:3.2    "/pause"                 36 minutes ago      Up 36 minutes                           k8s_POD_calico-node-h96l5_kube-system_74fa2a06-0378-4324-bd72-8332bdc5d4df_0
        5cf5795addd0        k8s.gcr.io/pause:3.2    "/pause"                 36 minutes ago      Up 36 minutes                           k8s_POD_kube-proxy-f2nfl_kube-system_98578f18-1a0d-4056-8fc5-fc4bd48ac1d1_0
        ```
    
3. Install ingress controller - [ref](https://kubernetes.github.io/ingress-nginx/deploy/#bare-metal)
    ```bash
    # Mandatory commands for all deployments
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.29.0/deploy/static/mandatory.yaml
 
    # Create a NodePort based ingress controller for bare-metal
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.29.0/deploy/static/provider/baremetal/service-nodeport.yaml
 
    # Verify installation
    kubectl get svc -n ingress-nginx 
    kubectl get pods --all-namespaces -l app.kubernetes.io/name=ingress-nginx --watch
    ```

4. Install loadbalancer - metallb - [ref](https://metallb.universe.tf/installation/)
    * Primary steps
        ```bash
        kubectl apply -f https://raw.githubusercontent.com/google/metallb/v0.9.3/manifests/namespace.yaml
        kubectl apply -f https://raw.githubusercontent.com/google/metallb/v0.9.3/manifests/metallb.yaml
        # On first install only
        kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
        ```
    * Apply layer2 config - [ref](https://kubernetes.github.io/ingress-nginx/deploy/baremetal/#a-pure-software-solution-metallb)
        ```bash
        apiVersion: v1
        kind: ConfigMap
        metadata:
          namespace: metallb-system
          name: config
        data:
          config: |
            address-pools:
            - name: default
              protocol: layer2
              addresses:
              - 10.11.16.80-10.11.16.85
        
        # the pool should be such that it doesn't include node ips otherwise
        ubuntu@k8s-master:~/kubernetes/configmap$ kubectl get svc -n ingress-nginx ingress-nginx  --watch
        NAME            TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
        ingress-nginx   LoadBalancer   10.108.15.101   <pending>     80:30213/TCP,443:31497/TCP   5h24m
  
        ubuntu@k8s-master:~/kubernetes/configmap$ kubectl get ing
        NAME                          CLASS    HOSTS   ADDRESS   PORTS   AGE
        kubernetes-bootcamp-ingress   <none>   *                 80      37m
        test-ingress                  <none>   *                 80      38m
        ```
    
    * Configure ingress-controller to use loadbalancer
        ```bash
        # Edit ingress-nginx svc from NodePort to LoadBalancer
        kubectl edit svc -n ingress-nginx ingress-nginx    
        
        ubuntu@k8s-master:~/kubernetes/configmap$ kubectl get svc -n ingress-nginx
        NAME            TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
        ingress-nginx   LoadBalancer   10.108.15.101   10.11.16.80   80:30213/TCP,443:31497/TCP   5h29m
  
        ubuntu@k8s-master:~/kubernetes/configmap$ kubectl get ing
        NAME                          CLASS    HOSTS   ADDRESS       PORTS   AGE
        kubernetes-bootcamp-ingress   <none>   *       10.11.16.80   80      42m
        test-ingress                  <none>   *       10.11.16.80   80      43m
        ```

5. Install kubernetes-dashboard - [ref](https://github.com/kubernetes/dashboard/blob/master/docs/user/installation.md#recommended-setup)
    * Create dashboard resource
        ```bash
        kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta8/aio/deploy/recommended.yaml
        ```
    * Create secret using certificates
        ```bash
        kubectl create secret generic kubernetes-dashboard-certs --from-file=$HOME/certs -n kubernetes-dashboard
        # or 
        kubectl create secret tls kubernetes-dashboard-certs --key="/home/ubuntu/certs/stg.key" --cert="/home/ubuntu/certs/stg.crt" -n kubernetes-dashboard
        
        # Sample output
        apiVersion: v1
        kind: Secret
        metadata:
          name: testsecret-tls
          namespace: ingress-nginx
        data:
          tls.crt: base64 encoded cert
          tls.key: base64 encoded key
        type: kubernetes.io/tls
        ```
    * Edit kubernetes-dashboard deployment to use certs
        ```bash
        kubectl edit deployment kubernetes-dashboard -n kubernetes-dashboard 
        ```
    * Create admin user - [ref](https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md)
        ```bash
        apiVersion: v1
        kind: ServiceAccount
        metadata:
          name: admin-user
          namespace: kubernetes-dashboard
        ---
        apiVersion: rbac.authorization.k8s.io/v1
        kind: ClusterRoleBinding
        metadata:
          name: kubernetes-dashboard-admin-user-cluster-role-binding
        roleRef:
          apiGroup: rbac.authorization.k8s.io
          kind: ClusterRole
          name: cluster-admin
        subjects:
        - kind: ServiceAccount
          name: admin-user
          namespace: kubernetes-dashboard
        ```
    * Create read-only-user (optional)
        ```bash
        apiVersion: v1
        kind: ServiceAccount
        metadata:
          name: view-only-user
          namespace: kubernetes-dashboard
        ---
        apiVersion: rbac.authorization.k8s.io/v1
        kind: ClusterRole
        metadata:
          name: dashboard-viewonly
        rules:
        - apiGroups:
          - ""
          resources:
          - configmaps
          - endpoints
          - persistentvolumeclaims
          - pods
          - replicationcontrollers
          - replicationcontrollers/scale
          - serviceaccounts
          - services
          - nodes
          - persistentvolumeclaims
          - persistentvolumes
          verbs:
          - get
          - list
          - watch
        - apiGroups:
          - ""
          resources:
          - bindings
          - events
          - limitranges
          - namespaces/status
          - pods/log
          - pods/status
          - replicationcontrollers/status
          - resourcequotas
          - resourcequotas/status
          verbs:
          - get
          - list
          - watch
        - apiGroups:
          - ""
          resources:
          - namespaces
          verbs:
          - get
          - list
          - watch
        - apiGroups:
          - apps
          resources:
          - daemonsets
          - deployments
          - deployments/scale
          - replicasets
          - replicasets/scale
          - statefulsets
          verbs:
          - get
          - list
          - watch
        - apiGroups:
          - autoscaling
          resources:
          - horizontalpodautoscalers
          verbs:
          - get
          - list
          - watch
        - apiGroups:
          - batch
          resources:
          - cronjobs
          - jobs
          verbs:
          - get
          - list
          - watch
        - apiGroups:
          - extensions
          resources:
          - daemonsets
          - deployments
          - deployments/scale
          - ingresses
          - networkpolicies
          - replicasets
          - replicasets/scale
          - replicationcontrollers/scale
          verbs:
          - get
          - list
          - watch
        - apiGroups:
          - policy
          resources:
          - poddisruptionbudgets
          verbs:
          - get
          - list
          - watch
        - apiGroups:
          - networking.k8s.io
          resources:
          - networkpolicies
          verbs:
          - get
          - list
          - watch
        - apiGroups:
          - storage.k8s.io
          resources:
          - storageclasses
          - volumeattachments
          verbs:
          - get
          - list
          - watch
        - apiGroups:
          - rbac.authorization.k8s.io
          resources:
          - clusterrolebindings
          - clusterroles
          - roles
          - rolebindings
          verbs:
          - get
          - list
          - watch
        ---
        apiVersion: rbac.authorization.k8s.io/v1
        kind: ClusterRoleBinding
        metadata:
          name: kubernetes-dashboard-view-only-user-cluster-role-binding
        roleRef:
          apiGroup: rbac.authorization.k8s.io
          kind: ClusterRole
          name: kubernetes-dashboard-viewonly-cluster-role
        subjects:
        - kind: ServiceAccount
          name: view-only-user
          namespace: kubernetes-dashboard
        ```
    * Get secret token for admin-user
        ```bash
        kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')
        ```
    * Expose dashboard outside
        * Create nodeport service
            ```bash
            kind: Service
            apiVersion: v1
            metadata:
              labels:
                k8s-app: kubernetes-dashboard
              name: kubernetes-dashboard-nodeport-svc
              namespace: kubernetes-dashboard
            spec:
              type: NodePort
              ports:
              - nodePort: 31532
                port: 8443
                targetPort: 8443
                protocol: TCP
              selector:
                k8s-app: kubernetes-dashboard
            ```
        * Create ingress
            * At root
                ```bash
                apiVersion: networking.k8s.io/v1beta1
                kind: Ingress
                metadata:
                  name: kubernetes-dashboard-ingress-tls
                  namespace: kubernetes-dashboard
                  annotations:
                    ingress.kubernetes.io/force-ssl-redirect: "true"
                    nginx.ingress.kubernetes.io/secure-backends: "true"
                    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
                    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
                spec:
                  tls:
                  - hosts:
                    - k8s.stg.blueleaf.com
                    secretName: kubernetes-dashboard-certs
                  rules:
                  - host: k8s.stg.blueleaf.com
                    http:
                      paths:
                      - backend:
                          serviceName: kubernetes-dashboard
                          servicePort: 443
                ```
            * At path - `/dashboard`
                ```bash
                apiVersion: networking.k8s.io/v1beta1
                kind: Ingress
                metadata:
                  name: kubernetes-dashboard-ingress-rewrite
                  namespace: kubernetes-dashboard
                  annotations:
                    nginx.ingress.kubernetes.io/rewrite-target: /$2
                    #ingress.kubernetes.io/force-ssl-redirect: "true"
                    nginx.ingress.kubernetes.io/secure-backends: "true"
                    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
                    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
                    nginx.ingress.kubernetes.io/configuration-snippet: rewrite ^(/dashboard)$ $1/ permanent;
                spec:
                  rules:
                    - http:
                        paths:
                        - path: "/dashboard(/|$)(.*)"
                          backend:
                            serviceName: kubernetes-dashboard
                            servicePort: 443
                ``` 

6. Install helm & tiller - [ref](https://v2.helm.sh/docs/using_helm/#installing-helm)
    * Install helm `v2.16.6` -
        * Run - `curl -L https://git.io/get_helm.sh | bash`
        * Verify - `helm version`
            ```bash
            # Sample output
            Client: &version.Version{SemVer:"v2.16.6", GitCommit:"dd2e5695da88625b190e6b22e9542550ab503a47", GitTreeState:"clean"}
            Error: could not find tiller
            ``` 
    * Install tiller `v2.16.6` -
        * Run - `helm init`
        * Verify - `helm version`
            ```bash
            # Sample output
            Client: &version.Version{SemVer:"v2.16.6", GitCommit:"dd2e5695da88625b190e6b22e9542550ab503a47", GitTreeState:"clean"}
            Server: &version.Version{SemVer:"v2.16.6", GitCommit:"dd2e5695da88625b190e6b22e9542550ab503a47", GitTreeState:"clean"}
            
            # tiller is installed in kube-system namespace - tiller-deploy-d47d5858d-6j6np
            ubuntu@k8s-master:~$ k get pods -n kube-system
            NAME                                          READY   STATUS    RESTARTS   AGE
            calico-kube-controllers-75d56dfc47-j4bcw      1/1     Running   5          26h
            calico-node-dk9t9                             1/1     Running   5          26h
            calico-node-h96l5                             1/1     Running   2          26h
            calico-node-pmbcp                             1/1     Running   2          26h
            coredns-66bff467f8-cr792                      1/1     Running   5          26h
            coredns-66bff467f8-x6dmj                      1/1     Running   5          26h
            etcd-k8s-master                      1/1     Running   5          26h
            kube-apiserver-k8s-master            1/1     Running   6          26h
            kube-controller-manager-k8s-master   1/1     Running   5          26h
            kube-proxy-98jjz                              1/1     Running   5          26h
            kube-proxy-f2nfl                              1/1     Running   2          26h
            kube-proxy-tbbhc                              1/1     Running   2          26h
            kube-scheduler-k8s-master            1/1     Running   5          26h
            tiller-deploy-d47d5858d-6j6np                 1/1     Running   0          55s
            ``` 
    * Create `serviceaccount` for tiller
        ```bash
        kubectl create serviceaccount -n kube-system tiller
        kubectl create clusterrolebinding tiller-cluster-admin --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
        
        # Otherwise following error will popup
        Error: configmaps is forbidden: User "system:serviceaccount:kube-system:default" cannot list resource "configmaps" in API group "" in the namespace "kube-system"
        ```

## Misc
1. Local docker registry
    * Setup - [ref](https://docs.docker.com/registry/deploying/)
        ```bash
        sudo docker run -d \
          --restart=always \
          --name registry \
          -v "$(pwd)"/certs:/certs \
          -v /data/registry:/var/lib/registry \
          -e REGISTRY_HTTP_ADDR=0.0.0.0:443 \
          -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
          -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
          -p 8080:443 \
          registry:2
        ```
    * Basic usage - [ref](https://docs.docker.com/registry/)
        ```bash
        # Start your registry
        docker run -d -p 5000:5000 --name registry registry:2

        # Pull (or build) some image from the hub
        docker pull ubuntu
  
        # Tag the image so that it points to your registry
        docker image tag ubuntu localhost:5000/myfirstimage
  
        # Push it
        docker push localhost:5000/myfirstimage
  
        # Pull it back
        docker pull localhost:5000/myfirstimage
  
        # Now stop your registry and remove all data
        docker container stop registry && docker container rm -v registry
        ```
    * List registry content
        ```bash
        # List repository
        curl https://<registry-server-ip>:8080/v2/_catalog
        
        # List tags of a repository
        curl https://<registry-server-ip>:8080/v2/<repo-name>/tags/list  
        ```
    * Delete repositories and tags of the images
        ```bash
        # Remove the repository or the manifests/tags to be deleted
        Removed dir /var/lib/registry/docker/registry/v2/repositories/[name]/_manifests/tags/[associatedTags]
  
        # Run the garbage collector to delete the images accordingly
        docker exec -it registry bin/registry garbage-collect /etc/docker/registry/config.yml -m
        ```  

## Administration
1. Enable kubectl autocompletion
    ```bash
    vim ~/.bashrc
       source <(kubectl completion bash)
       alias k=kubectl
       complete -F __start_kubectl k  
    ```
    
2. General Resource query - [ref](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
    * `kubectl get [resourceTypes] -l [label]=[value] -n [nameSpaces] -o wide`
    * Resources:
        ```bash
          * certificatesigningrequests (aka 'csr')
          * clusterrolebindings
          * clusterroles
          * componentstatuses (aka 'cs')
          * configmaps (aka 'cm')
          * controllerrevisions
          * cronjobs
          * customresourcedefinition (aka 'crd')
          * daemonsets (aka 'ds')
          * deployments (aka 'deploy')
          * endpoints (aka 'ep')
          * events (aka 'ev')
          * horizontalpodautoscalers (aka 'hpa')
          * ingresses (aka 'ing')
          * jobs
          * limitranges (aka 'limits')
          * namespaces (aka 'ns')
          * networkpolicies (aka 'netpol')
          * nodes (aka 'no')
          * persistentvolumeclaims (aka 'pvc')
          * persistentvolumes (aka 'pv')
          * poddisruptionbudgets (aka 'pdb')
          * podpreset
          * pods (aka 'po')
          * podsecuritypolicies (aka 'psp')
          * podtemplates
          * replicasets (aka 'rs')
          * replicationcontrollers (aka 'rc')
          * resourcequotas (aka 'quota')
          * rolebindings
          * roles
          * secrets
          * serviceaccounts (aka 'sa')
          * services (aka 'svc')
          * statefulsets (aka 'sts')
          * storageclasses (aka 'sc')
        ```

3. Get IP Range
    * Services IP range: ```kubectl cluster-info dump | grep -m 1 service-cluster-ip-range```
    * Pods IP range: ```kubectl cluster-info dump | grep -m 1 cluster-cidr```
    * With kubeadm: 
        ```
        kubeadm config view | grep Subnet
        Output:
        podSubnet: 10.10.0.0/16
        serviceSubnet: 10.96.0.0/12
        ```
        
4. Detach a node out from cluster -
    * On control-panel
        ```bash
        kubectl drain <node name> --delete-local-data --force --ignore-daemonsets
        kubectl cordon <node name>
        kubectl delete node <node name>
        ```
    * On node to be deleted
        ```bash
        kubeadm reset
        ```
        
5. Disaster recovery
    ```bash
    # Remove manifests in kubernetes directory on control-panel node (if the same node is being reset)
    sudo rm -rf /etc/kubernetes/manifests
 
    # Bootstrap control-panel using kubeadm with the same etcd data directory
    sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --ignore-preflight-errors=DirAvailable--var-lib-etcd]
    
    # Join the worker nodes with these new sets of certs and keys
    ...
    ```

6. Completely uninstall kubernetes cluster and its components
    ```bash
    kubeadm reset
    sudo apt-get purge kubeadm kubectl kubelet kubernetes-cni kube*   
    sudo apt-get autoremove  
    sudo rm -rf ~/.kube
    ```

## Testing the setup
1. Create sample deployment - `kubectl create deployment kubernetes-bootcamp --image=gcr.io/google-samples/kubernetes-bootcamp:v1`

2. Create nodeport service -
    ```bash
    apiVersion: v1
    kind: Service
    metadata:
      name: kubernetes-bootcamp
      namespace: default
    spec:
      type: NodePort
      ports:
      - nodePort: 31341
        port: 8082
        targetPort: 8080
        protocol: TCP
      selector:
        app: kubernetes-bootcamp
    ``` 
    
3. Create ingress -
    ```bash
    apiVersion: networking.k8s.io/v1beta1
    kind: Ingress
    metadata:
      name: kubernetes-bootcamp-ingress
      #annotations:
      # nginx.ingress.kubernetes.io/rewrite-target: /
    spec:
      rules:
      - http:
          paths:
          - path: /testpath
            backend:
              serviceName: kubernetes-bootcamp
              servicePort: 8082
    ```

## References
* etcd
    * https://medium.com/better-programming/a-closer-look-at-etcd-the-brain-of-a-kubernetes-cluster-788c8ea759a5
* Ingress
    * https://docs.nginx.com/nginx-ingress-controller/installation/installation-with-manifests/
    * https://github.com/kubernetes/ingress-nginx/tree/master/docs/examples/rewrite* https://github.com/kubernetes/ingress-nginx/tree/master/docs/examples/rewrite
    * https://oteemo.com/2018/11/26/wild-card-certificates-with-nginx-ingress-and-kubernetes-cronjobs/
    * https://github.com/nginxinc/kubernetes-ingress/tree/master/examples/wildcard-tls-certificate
    * https://kubernetes.github.io/ingress-nginx/user-guide/tls/
* Disaster recovery
    * https://medium.com/velotio-perspectives/the-ultimate-guide-to-disaster-recovery-for-your-kubernetes-clusters-94143fcc8c1e