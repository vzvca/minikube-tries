# minikube-tries

Minikube gists and notes. Used to remember how to do things. Maybe useful for others who knows ...

## World map

This tutorial shows :
  * how to use minikube builtin registry,
  * how to build a really minimal docker container using a statically linked ELF executable,
  * how to push this image to minkibe registry,
  * how to create a deployment using previous image,
  * how to 'expose' this deployment. 

In this minikube tutorial, minikube uses the docker driver.

    $ minikube start
    $ minikube addons enable registry

With the docker driver `docker ps` contains a line with the minikube control plane :

    CONTAINER ID   IMAGE                                 COMMAND                  ...
    b67aa409e890   gcr.io/k8s-minikube/kicbase:v0.0.29   "/usr/local/bin/entr…"   ...


The supplied executable is a statically linked C program which runs an http server.
See "https://github.com/vzvca/mbtiles-offline-viewer/tree/main/stage1".
The program was linked with `cc -static -o world <objects> <libs>` instead of normal link.

First create an empty scratch image :

$ tar cv --files-from /dev/null | docker import - scratch

The scratch image is used as a base to build a minimalist container :

    $ cat Dockerfile
    FROM scratch
    ADD world /world
    CMD ["/world","-p","9001"]

The container will run the world web server listening on port 9001.

build image

    $ docker build -t world:0.1 .
    Sending build context to Docker daemon  8.562MB
    Step 1/3 : FROM scratch
     ---> 
    Step 2/3 : ADD world /world
     ---> Using cache
     ---> 95af1433d980
    Step 3/3 : CMD ["/world","-p","9001"]
     ---> Using cache
     ---> 744630f11b20
    Successfully built 744630f11b20
    Successfully tagged world:0.1

Push image to minikube registry

    $ docker tag world:0.1 localhost:5000/world:0.1

in another window

    $ alias kubectl='minikube kubectl --'

    $ kubectl port-forward --namespace kube-system service/registry 5000:80
    Forwarding from 127.0.0.1:5000 -> 5000
    Forwarding from [::1]:5000 -> 5000

    $ docker push localhost:5000/world:0.1
    The push refers to repository [localhost:5000/world]
    6a72bc64fabe: Pushed 
    0.1: digest: sha256:ca361fa5d62f0f00bf04529ccd5d5b4ce531350e846a20bbd05846268e461dc9 size: 528

    $ docker rmi localhost:5000/world:0.1
    Untagged: localhost:5000/world:0.1
    Untagged: localhost:5000/world@sha256:ca361fa5d62f0f00bf04529ccd5d5b4ce531350e846a20bbd05846268e461dc9

The image can now be used from the minikube registry :

    $ minikube ssh
    docker@minikube:~$ docker pull localhost:5000/world:0.1
    0.1: Pulling from world
    39463436ea17: Pull complete 
    Digest: sha256:ca361fa5d62f0f00bf04529ccd5d5b4ce531350e846a20bbd05846268e461dc9
    Status: Downloaded newer image for localhost:5000/world:0.1
    localhost:5000/world:0.1

    docker@minikube:~$ docker images
    REPOSITORY                                     TAG       IMAGE ID       CREATED          SIZE
    localhost:5000/world                           0.1       744630f11b20   41 minutes ago   8.56MB
    k8s.gcr.io/kube-apiserver                      v1.23.1   b6d7abedde39   8 weeks ago      135MB
    k8s.gcr.io/kube-proxy                          v1.23.1   b46c42588d51   8 weeks ago      112MB
    k8s.gcr.io/kube-controller-manager             v1.23.1   f51846a4fd28   8 weeks ago      125MB
    k8s.gcr.io/kube-scheduler                      v1.23.1   71d575efe628   8 weeks ago      53.5MB
    k8s.gcr.io/etcd                                3.5.1-0   25f8c7f3da61   3 months ago     293MB
    k8s.gcr.io/coredns/coredns                     v1.8.6    a4ca41631cc7   4 months ago     46.8MB
    k8s.gcr.io/pause                               3.6       6270bb605e12   5 months ago     683kB
    kubernetesui/dashboard                         v2.3.1    e1482a24335a   8 months ago     220MB
    kubernetesui/metrics-scraper                   v1.0.7    7801cfc6d5c0   8 months ago     34.4MB
    gcr.io/k8s-minikube/storage-provisioner        v5        6e38f40d628d   10 months ago    31.5MB
    registry                                       <none>    678dfa38fcfa   14 months ago    26.2MB
    gcr.io/google_containers/kube-registry-proxy   <none>    60dc18151daf   5 years ago      188MB

Create a deployment using the image

    $ kubectl create deployment world --image localhost:5000/world:0.1
    deployment.apps/world created
    $ kubectl get pods
    NAME                     READY   STATUS    RESTARTS   AGE
    world-5db997cf5d-9j5hr   1/1     Running   0          10s

    $ kubectl expose deployment world --port=9001
    service/world exposed
    $ kubectl get services
    NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
    kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP    18m
    world        ClusterIP   10.104.89.201   <none>        9001/TCP   3s

    $ kubectl port-forward --namespace default service/world 9001:9001
    Forwarding from 127.0.0.1:9001 -> 9001
    Forwarding from [::1]:9001 -> 9001

Now open http://127.0.0.1:9001 in a browser.
Note that you have to use the same port 9001 inside and outside of the container.

