# UPI instalation

Diretorio de instalação será /root/install/

01. prepara dns (verificação dos dominios)
02. prepara dhcp (verificação do ip fixados aos macs)
03. prepara PXE 
04. prepara balancer (haproxy, ajustar os endereços IPs)
05. prepara firewall do bastion
06. Download clients (oc e openshift-install na versão do cluster)
07. Gerar ignitions
    * openshift-install create manifests --dir install
    * openshift-install create ignition-configs --dir install
08. Configurações dos containers
    * Configurar container HTTPD
    * Configurar container PXE
    * Configurar container haproxy
09. Iniciar a instalação
    * openshift-install wait-for bootstrap-complete --dir install
10. Aprovações dos nodes no cluster 
    * oc adm certificate approve `oc get csr | grep Pending |awk '{print $1}'`
    * execute ./approve_crs.sh 
11. Verificação da conclusão da instalação 
    * openshift-install wait-for install-complete --dir=install --log-level=debug
12. Verificação do cluster
    * oc get co

## Firewall
* firewall-cmd --permanent --zone=public --add-service=tftp 
* firewall-cmd --permanent --zone=public --add-port=9090/tcp 
* firewall-cmd --permanent --zone=public --add-port=9091/tcp 
* firewall-cmd --permanent --zone=public --add-port=443/tcp
* firewall-cmd --permanent --zone=public --add-port=32700/tcp
* firewall-cmd --permanent --zone=public --add-port=22623/tcp 
* firewall-cmd --permanent --zone=public --add-port=6443/tcp 
* firewall-cmd --permanent --zone=public --add-port=80/tcp 
* firewall-cmd --permanent --zone=public --add-port=10250/tcp 
* firewall-cmd --add-service=tftp --permanent
* firewall-cmd --reload
* firewall-cmd --list-all

## Download oc clients
* scp /home/lagomes/MEGA/openshift/Instalação-UPI/client.sh root@10.36.17.1:~/client.sh
* ./client.sh

## Copy install config
* scp /home/lagomes/MEGA/openshift/Instalação-UPI/install-config.yaml root@10.36.17.1:~/install/install-config.yaml

### Deploy container of PXE-Menu boot file container
* ./images_download.sh

* scp /home/lagomes/MEGA/openshift/Instalação-UPI/pxelinux.cfg/* root@10.36.17.1:/root/pxelinux.cfg/

* mkdir /root/pxelinux.cfg
* chmod -R +r /root/pxelinux.cfg/*

* podman run -d --restart=always --name tftp-server -p69:69/udp \
    -v /root/pxelinux.cfg:/var/lib/tftpboot/pxelinux.cfg/:Z \
    -v /root/coreos/:/var/lib/tftpboot/pxelinux/images/coreos/:Z \
    quay.io/lagomes/pxe-tftp:latest

* podman run -d --restart=always --name tftp-server quay.io/lagomes/pxe-tftp:latest /bin;bash 

## Deploy container of HTTPD ignitions file

* mkdir /root/install/
* chmod -R +r /root/install/*

* podman run -d --name httpd --restart=always \
    -v /root/install/:/usr/local/apache2/htdocs/:Z \
    -p9090:80 \
    httpd:latest

* para testar acesse http://10.36.17.1:9090/bootstrap.ign

### Download files for boot (não utilizado)
* ./images_download.sh

* podman run -d --name httpd-images --restart=always -v /root/coreos/:/usr/local/apache2/htdocs/:Z -p9091:80 httpd:latest

* para testar acesse http://10.36.17.1:9091/

## Deploy container of Haproxy for LB

* scp /home/lagomes/MEGA/openshift/Instalação-UPI/haproxy/* root@10.36.17.1:/root/haproxy/

* podman run -d --name haproxy --restart=always \
    -v /root/haproxy:/usr/local/etc/haproxy:Z \
    -p 80:80 \
    -p 443:443 \
    -p 22623:22623 \
    -p 6443:6443 \
    -p 32700:32700 \
    haproxytech/haproxy-alpine:2.4

## restart podmans
* podman restart httpd
* podman restart tftp-server
* podman restart haproxy
* podman restart httpd-images

# Continuidade da intalação do cluster
* export KUBECONFIG=/root/install/auth/kubeconfig

## Roles
* oc label nodes infra01.lagomes.rhbr-lab.com node-role.kubernetes.io/infra=
* oc label nodes infra02.lagomes.rhbr-lab.com node-role.kubernetes.io/infra=
* oc label nodes infra03.lagomes.rhbr-lab.com node-role.kubernetes.io/infra=
* oc label nodes infra01.lagomes.rhbr-lab.com node-role.kubernetes.io/worker-
* oc label nodes infra02.lagomes.rhbr-lab.com node-role.kubernetes.io/worker-
* oc label nodes infra03.lagomes.rhbr-lab.com node-role.kubernetes.io/worker-

## Taint
*  oc adm taint nodes infra01.lagomes.rhbr-lab.com infra=reserved:NoExecute
*  oc adm taint nodes infra02.lagomes.rhbr-lab.com infra=reserved:NoExecute
*  oc adm taint nodes infra03.lagomes.rhbr-lab.com infra=reserved:NoExecute
*  oc adm taint nodes infra01.lagomes.rhbr-lab.com infra=reserved:NoSchedule
*  oc adm taint nodes infra02.lagomes.rhbr-lab.com infra=reserved:NoSchedule
*  oc adm taint nodes infra03.lagomes.rhbr-lab.com infra=reserved:NoSchedule

## Router
* oc patch ingresscontroller/default -n  openshift-ingress-operator  --type=merge -p '{"spec":{"nodePlacement": {"nodeSelector": {"matchLabels": {"node-role.kubernetes.io/infra": ""}},"tolerations": [{"effect":"NoSchedule","key": "infra","value": "reserved"},{"effect":"NoExecute","key": "infra","value": "reserved"}]}}}'

* oc patch ingresscontroller/default -n  openshift-ingress-operator  --type=merge -p '{"spec":{"nodePlacement": {"nodeSelector": {"matchLabels": {"node-role.kubernetes.io/infra": ""}},"tolerations": [{"effect":"NoSchedule","key": "node-role.kubernetes.io/infra","operator": "Exists"}]}}}'

* oc patch ingresscontroller/default -n   openshift-ingress-operator --type=merge -p '{"spec":{"replicas": 3}}'

## Configure authentication
* scp -i /home/lagomes/.ssh/id_ed25519_redhat  root@10.36.17.1:install/auth/kubeconfig /home/lagomes/.kube/config
* oc apply -k /home/lagomes/MEGA/openshift/gitops-gitops/


