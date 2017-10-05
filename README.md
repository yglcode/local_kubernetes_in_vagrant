local kubernetes+flannel cluster with vagrant + kubeadm
=======================================================

1. copy this Vagrantfile to your project directory
2. add more worker nodes by changing node_num in Vagrantfile
3. run "vagrant up":
       deb packages and docker images will be downloaded, installed and cached in "cache" directory
4. later reinstall or install for different projects will install from local cache

vagrant ssh into master1, node1 or other nodes; run k8s CLI: "kubectl get nodes",...



