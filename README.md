# Performing a K8s Upgrade with kubeadm

Current version 1.23.0
Moving to 1.23.2


### Get current kubeadm Version

Use this on control plane and worker nodes to check the current kubeadm version installed.
```
$ kubeadm version
```
```
kubeadm version: &version.Info{Major:"1", Minor:"23", GitVersion:"v1.23.0", GitCommit:"ab69524f795c42094a6630298ff53f3c3ebab7f4", GitTreeState:"clean", BuildDate:"2021-12-07T18:15:11Z", GoVersion:"go1.17.3", Compiler:"gc", Platform:"linux/amd64"}
```


### Get Control Plane and Worker Nodes Names
```
$ kubectl get nodes
```
```
NAME               STATUS   ROLES                  AGE     VERSION
ip-10-101-32-196   Ready    <none>                 5h57m   v1.23.0
ip-10-101-33-147   Ready    control-plane,master   6h2m    v1.23.0
ip-10-101-35-71    Ready    <none>                 5h57m   v1.23.0
```

### 1) Upgrade Control Plane

Upgrade kubeadm:
```
[cloud_user@k8s-control]$ sudo apt-get update && \
sudo apt-get install -y --allow-change-held-packages kubeadm=1.23.2-00
```

Make sure it upgraded correctly:
```
[cloud_user@k8s-control]$ kubeadm version
>kubeadm version: &version.Info{Major:"1", Minor:"23",
GitVersion:"v1.23.2",
GitCommit:"9d142434e3af351a628bffee3939e64c681afa4d",
GitTreeState:"clean", BuildDate:"2022-01-19T17:34:34Z",
GoVersion:"go1.17.5", Compiler:"gc", Platform:"linux/amd64"}
```

Drain Control Plane node:
```
$ kubectl drain ip-10-101-33-147 --ignore-daemonsets
```
```
node/ip-10-101-33-147 cordoned
WARNING: ignoring DaemonSet-managed Pods: kube-system/calico-node-49t4r, kube-system/kube-proxy-7vhgj
evicting pod kube-system/coredns-64897985d-zq9f7
evicting pod kube-system/calico-kube-controllers-7bc6547ffb-l5nb8
evicting pod kube-system/coredns-64897985d-6k9k7
pod/calico-kube-controllers-7bc6547ffb-l5nb8 evicted
pod/coredns-64897985d-zq9f7 evicted
pod/coredns-64897985d-6k9k7 evicted
node/ip-10-101-33-147 drained
```

Plan the upgrade:
```
[cloud_user@k8s-control]$ sudo kubeadm upgrade plan v1.23.2
```

```
[upgrade/config] Making sure the configuration is correct:
[upgrade/config] Reading configuration from the cluster...
[upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[preflight] Running pre-flight checks.
[upgrade] Running cluster health checks
[upgrade] Fetching available versions to upgrade to
[upgrade/versions] Cluster version: v1.23.0
[upgrade/versions] kubeadm version: v1.23.2
[upgrade/versions] Target version: v1.23.2
[upgrade/versions] Latest version in the v1.23 series: v1.23.2

Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
COMPONENT   CURRENT       TARGET
kubelet     3 x v1.23.0   v1.23.2

Upgrade to the latest version in the v1.23 series:

COMPONENT                 CURRENT   TARGET
kube-apiserver            v1.23.0   v1.23.2
kube-controller-manager   v1.23.0   v1.23.2
kube-scheduler            v1.23.0   v1.23.2
kube-proxy                v1.23.0   v1.23.2
CoreDNS                   v1.8.6    v1.8.6
etcd                      3.5.1-0   3.5.1-0

You can now apply the upgrade by executing the following command:

	kubeadm upgrade apply v1.23.2

_____________________________________________________________________


The table below shows the current state of component configs as understood by this version of kubeadm.
Configs that have a "yes" mark in the "MANUAL UPGRADE REQUIRED" column require manual config upgrade or
resetting to kubeadm defaults before a successful upgrade can be performed. The version to manually
upgrade to is denoted in the "PREFERRED VERSION" column.

API GROUP                 CURRENT VERSION   PREFERRED VERSION   MANUAL UPGRADE REQUIRED
kubeproxy.config.k8s.io   v1alpha1          v1alpha1            no
kubelet.config.k8s.io     v1beta1           v1beta1             no
_____________________________________________________________________
```

Upgrade the control plane components:
```
[cloud_user@k8s-control]$ sudo kubeadm upgrade apply v1.23.2
```
```
[upgrade/config] Making sure the configuration is correct:
[upgrade/config] Reading configuration from the cluster...
[upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
W0704 16:04:53.377623  214029 utils.go:69] The recommended value for "resolvConf" in "KubeletConfiguration" is: /run/systemd/resolve/resolv.conf; the provided value is: /run/systemd/resolve/resolv.conf
[preflight] Running pre-flight checks.
[upgrade] Running cluster health checks
[upgrade/version] You have chosen to change the cluster version to "v1.23.2"
[upgrade/versions] Cluster version: v1.23.0
[upgrade/versions] kubeadm version: v1.23.2
[upgrade/confirm] Are you sure you want to proceed with the upgrade? [y/N]: y
...
[upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.23.2". Enjoy!
...
[upgrade/kubelet] Now that your control plane is upgraded, please proceed with upgrading your kubelets if you haven't already done so.
```

Upgrade kubelet and kubectl on the control plane node:
```
[cloud_user@k8s-control]$ sudo apt-get update && \
sudo apt-get install -y --allow-change-held-packages kubelet=1.23.2-00 kubectl=1.23.2-00
```
```
Hit:1 http://repomirror.sjc.dsinternal.org/ubuntu focal
...
Reading state information... Done
The following held packages will be changed:
  kubectl kubelet
The following packages will be upgraded:
  kubectl kubelet
2 upgraded, 0 newly installed, 0 to remove and 224 not upgraded.
...
Unpacking kubelet (1.23.2-00) over (1.23.0-00) ...
Setting up kubectl (1.23.2-00) ...
Setting up kubelet (1.23.2-00) ...
```

Restart kubelet:
```
[cloud_user@k8s-control]$ sudo systemctl daemon-reload
```
```
[cloud_user@k8s-control]$ sudo systemctl restart kubelet
```

Uncordon the control plane node:
```
[cloud_user@k8s-control]$ kubectl uncordon ip-10-101-33-147
```

```
node/ip-10-101-33-147 uncordoned
```


Verify the control plane is working:
```
[cloud_user@k8s-control]$ kubectl get nodes
```
```
NAME               STATUS   ROLES                  AGE     VERSION
ip-10-101-32-196   Ready    <none>                 6h27m   v1.23.0
ip-10-101-33-147   Ready    control-plane,master   6h32m   v1.23.2
ip-10-101-35-71    Ready    <none>                 6h27m   v1.23.0
```


Note control plane nodes has been updated as version has changed. If it shows a NotReady status, run the command again after a minute or so, it should become Ready.


### 2) Upgrade Working Nodes

#### Worker Node 1
Get node information again
```
$ kubectl get nodes
```
```
NAME               STATUS   ROLES                  AGE     VERSION
ip-10-101-32-196   Ready    <none>                 7h7m    v1.23.0
ip-10-101-33-147   Ready    control-plane,master   7h12m   v1.23.2
ip-10-101-35-71    Ready    <none>                 7h6m    v1.23.0
```

Run the following on the control plane node to drain worker node 1:
```
[cloud_user@k8s-control]$ kubectl drain ip-10-101-32-196 --ignore-daemonsets --force
```
You may get an error message that certain pods couldn't be deleted, which is fine.
```
node/ip-10-101-32-196 cordoned
WARNING: ignoring DaemonSet-managed Pods: kube-system/calico-node-28hqv, kube-system/kube-proxy-kftjr
evicting pod kube-system/coredns-64897985d-kq4r5
evicting pod kube-system/calico-kube-controllers-7bc6547ffb-6x699
pod/calico-kube-controllers-7bc6547ffb-6x699 evicted
pod/coredns-64897985d-kq4r5 evicted
node/ip-10-101-32-196 drained
```

In a new terminal window, log in to worker node 1:

```
$ ssh cloud_user@10.101.32.196
```

Upgrade kubeadm on worker node 1:
```
[cloud_user@k8s-worker1]$ sudo apt-get update && \
sudo apt-get install -y --allow-change-held-packages kubeadm=1.23.2-00
```
```
Hit:1 http://repomirror.sjc.dsinternal.org/ubuntu focal InRelease
Get:2 http://repomirror.sjc.dsinternal.org/ubuntu focal-updates InRelease [114 kB]
Get:3 http://repomirror.sjc.dsinternal.org/ubuntu focal-backports InRelease [108 kB]
Get:4 http://repomirror.sjc.dsinternal.org/ubuntu focal-security InRelease [114 kB]
Get:6 http://repomirror.sjc.dsinternal.org/ubuntu focal-updates/main amd64 Packages [1946 kB]
Get:7 http://repomirror.sjc.dsinternal.org/ubuntu focal-updates/main amd64 c-n-f Metadata [15.6 kB]
Get:8 http://repomirror.sjc.dsinternal.org/ubuntu focal-updates/universe amd64 Packages [924 kB]
Get:9 http://repomirror.sjc.dsinternal.org/ubuntu focal-updates/universe amd64 c-n-f Metadata [20.9 kB]
Get:10 http://repomirror.sjc.dsinternal.org/ubuntu focal-updates/multiverse amd64 Packages [24.5 kB]
Get:11 http://repomirror.sjc.dsinternal.org/ubuntu focal-updates/multiverse amd64 c-n-f Metadata [592 B]
Hit:5 https://packages.cloud.google.com/apt kubernetes-xenial InRelease
Get:12 http://repomirror.sjc.dsinternal.org/ubuntu focal-security/main amd64 Packages [1589 kB]
Get:13 http://repomirror.sjc.dsinternal.org/ubuntu focal-security/main amd64 c-n-f Metadata [10.6 kB]
Fetched 4868 kB in 1s (3900 kB/s)
Reading package lists... Done
Reading package lists... Done
Building dependency tree
Reading state information... Done
The following held packages will be changed:
  kubeadm
The following packages will be upgraded:
  kubeadm
1 upgraded, 0 newly installed, 0 to remove and 226 not upgraded.
Need to get 8580 kB of archives.
After this operation, 8192 B of additional disk space will be used.
Get:1 https://packages.cloud.google.com/apt kubernetes-xenial/main amd64 kubeadm amd64 1.23.2-00 [8580 kB]
Fetched 8580 kB in 2s (3764 kB/s)
(Reading database ... 48077 files and directories currently installed.)
Preparing to unpack .../kubeadm_1.23.2-00_amd64.deb ...
Unpacking kubeadm (1.23.2-00) over (1.23.0-00) ...
Setting up kubeadm (1.23.2-00) ...
```
Check the new kubeadm version on worker node 1
```
[cloud_user@k8s-worker1]$ kubeadm version
```
```
kubeadm version: &version.Info{Major:"1", Minor:"23", GitVersion:"v1.23.2", GitCommit:"9d142434e3af351a628bffee3939e64c681afa4d", GitTreeState:"clean", BuildDate:"2022-01-19T17:34:34Z", GoVersion:"go1.17.5", Compiler:"gc", Platform:"linux/amd64"}
```

Upgrade kubeadm in worker node 1

```
[cloud_user@k8s-worker1]$ sudo kubeadm upgrade node
```
```
[upgrade] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
W0704 17:06:08.918143  164497 utils.go:69] The recommended value for "resolvConf" in "KubeletConfiguration" is: /run/systemd/resolve/resolv.conf; the provided value is: /run/systemd/resolve/resolv.conf
[preflight] Running pre-flight checks
[preflight] Skipping prepull. Not a control plane node.
[upgrade] Skipping phase. Not a control plane node.
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[upgrade] The configuration for this node was successfully updated!
[upgrade] Now you should go ahead and upgrade the kubelet package using your package manager.
```


Upgrade kubelet and kubectl on worker node 1:
```
[cloud_user@k8s-worker1]$ sudo apt-get update && \
sudo apt-get install -y --allow-change-held-packages kubelet=1.23.2-00 kubectl=1.23.2-00
```
```
Hit:1 http://repomirror.sjc.dsinternal.org/ubuntu focal InRelease
...
The following held packages will be changed:
  kubectl kubelet
The following packages will be upgraded:
  kubectl kubelet
...
Preparing to unpack .../kubectl_1.23.2-00_amd64.deb ...
Unpacking kubectl (1.23.2-00) over (1.23.0-00) ...
Preparing to unpack .../kubelet_1.23.2-00_amd64.deb ...
Unpacking kubelet (1.23.2-00) over (1.23.0-00) ...
Setting up kubectl (1.23.2-00) ...
Setting up kubelet (1.23.2-00) ...
```

Restart kubelet in worker node 1:
```
[cloud_user@k8s-worker1]$ sudo systemctl daemon-reload
```
```
[cloud_user@k8s-worker1]$ sudo systemctl restart kubelet
```

From the control plane node, uncordon worker node 1:
```
[cloud_user@k8s-control]$ kubectl uncordon ip-10-101-32-196.srv101.dsinternal.org
```
```
node/ip-10-101-32-196.srv101.dsinternal.org uncordoned
```
####  For the subsequents worker nodes, do the same steps following the part 2 of this cookbook, "Upgrade Working Nodes".

### 3) Verify K8S cluster has been upgraded

In the control plane node, verify the cluster is upgraded and working:
```
[cloud_user@k8s-control]$ kubectl get nodes
```
```
NAME                                     STATUS   ROLES                  AGE     VERSION
ip-10-101-32-196.srv101.dsinternal.org   Ready    <none>                 7h34m   v1.23.2
ip-10-101-33-147.srv101.dsinternal.org   Ready    control-plane,master   7h39m   v1.23.2
ip-10-101-35-71.srv101.dsinternal.org    Ready    <none>                 7h34m   v1.23.2
```

If they show a NotReady status, run the command again after a minute or so. They should become Ready.
