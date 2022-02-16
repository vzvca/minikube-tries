# Persistent volume

This tutorial shows :
  * Use of persistent volume,
  * Creation persistent volume that survive minikube stop/start,
  * Running eally crude web server using busybox image with web content in a PV,
  * Copy data to the the host and the PV persistent storage,
  * Expose the web server on local machine.

## Creating 'persistent' PV

With the following commands, create a directory inside minikube node and copy the web site content to it.

    $ minikube start
    $ minikube ssh
    docker@minikube $ mkdir /data/www
    docker@minikube $ exit
    $ minikube copy hello.txt /data/www/hello.txt
    $ minikube ssh ls /data/www
    hello.txt

Check that the content of `/data/www` survives a minikube shutdown.

    $ minikube stop 
    $ minikube start
    $ minikube ssh ls /data/www
    hello.txt


## Creating the minikube application

A single yaml file contains everything to deploy the application.
Use the following command to deploy the web server app in minikube cluster :

    $ kubectl apply -f todo.yml

You can check that everything has been created :

    $ kubectl get pv
    NAME     CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM             STORAGECLASS   REASON   AGE
    www-pv   32k        RWO            Retain           Bound    default/www-pvc   www-stc                 78m
    $ kubectl get pvc
    NAME      STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
    www-pvc   Bound    www-pv   32k        RWO            www-stc        78m
    $ kubectl get deployment
    NAME         READY   UP-TO-DATE   AVAILABLE   AGE
    busy-httpd   1/1     1            1           36m
    $ kubectl get pod
    NAME                          READY   STATUS    RESTARTS   AGE
    busy-httpd-6dc59545db-6w6bg   1/1     Running   0          36m
    $ kubectl get services
    NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
    busy-httpd   NodePort    10.108.194.50   <none>        8888:30143/TCP   20m
    kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP          116m

The service is of type NodePort, get its URL and open it in a browser :

    $ minikube service list
    |----------------------|------------------------------------|--------------|---------------------------|
    |      NAMESPACE       |                NAME                | TARGET PORT  |            URL            |
    |----------------------|------------------------------------|--------------|---------------------------|
    | default              | busy-httpd                         |         8888 | http://192.168.49.2:30143 |
    | default              | kubernetes                         | No node port |
    | ingress-nginx        | ingress-nginx-controller           | http/80      | http://192.168.49.2:31436 |
    |                      |                                    | https/443    | http://192.168.49.2:31919 |
    | ingress-nginx        | ingress-nginx-controller-admission | No node port |
    | kube-system          | kube-dns                           | No node port |
    | kube-system          | registry                           | No node port |
    | kubernetes-dashboard | dashboard-metrics-scraper          | No node port |
    | kubernetes-dashboard | kubernetes-dashboard               | No node port |
    |----------------------|------------------------------------|--------------|---------------------------|
    
    $ xdg-open http://192.168.49.2:30143/hello.txt

The URL can be guessed it is `http://$(minikube ip):<svc-port>/`, `svc-port` is read from column `PORT(S)` of output of `kubectl get services` (in this case `8888:30143/TCP`).

The content of `hello.txt` should display in your browser's window.

## Ingress

    $ kubectl apply -f ingress.yml 
    ingress.networking.k8s.io/busy-httpd created
    $ kubectl get ingress
    NAME         CLASS   HOSTS             ADDRESS   PORTS   AGE
    busy-httpd   nginx   busy-httpd.info             80      5s



    


