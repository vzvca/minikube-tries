# UDP communication between pods

## Deploy kubernetes application

Create kubernetes application in minikube cluster :

     $ kubectl apply -f todo.yml
     
It will create two deployments based on busybox and one service:
  * deployment `udpsvd` : starts `nc` listening on UDP port 9999. Upon connection, forks `tee -a /tmp/recv` which outputs incoming data to `/tmp/recv`.
  * deployment `shell` : sleeps for 1 day. This pod is meant to be used interactively by running `/bin/sh` in it.
  * service `udsvd` : exposes UDP:9999 port of deployement `udpsvd` to the cluster. DNS name for this service is `udpsvd.default.svc.cluster.local`.

## Testing UDP communication

Two terminals are needed from there.

In 1st terminal, open a shell in pod of deployment `shell` and start entering text. After each newline, go to the second window (see below) :

     $ kubectl exec shell-886f65646-qbsfx -ti -- /bin/sh
     / # echo encore | nc -u udpsvd.default.svc.cluster.local 9999
     / # nc -u udpsvd.default.svc.cluster.local 9999
     hello
     hello
     Is there somebody here !!
     Is there somebody here !!
     the communication is established.
     the communication is established.
     I will leave now.
     I will leave now.
     ^Cpunt!

     / # exit
     command terminated with exit code 130

In the second window, after each line entered, execute command `cat /tmp/recv` in `udpsvd` pod:

     $ kubectl exec udpsvd-58cbdc788d-klnwp -ti -- cat /tmp/recv
     hello
     $ kubectl exec udpsvd-58cbdc788d-klnwp -ti -- cat /tmp/recv
     hello
     Is there somebody here !!
     $ kubectl exec udpsvd-58cbdc788d-klnwp -ti -- cat /tmp/recv
     hello
     Is there somebody here !!
     the communication is established.
     $ kubectl exec udpsvd-58cbdc788d-klnwp -ti -- cat /tmp/recv
     hello
     Is there somebody here !!
     the communication is established.
     $ kubectl exec udpsvd-58cbdc788d-klnwp -ti -- cat /tmp/recv
     hello
     Is there somebody here !!
     the communication is established.
     I will leave now.

You can see the logs tracing the connection :

     $ kubectl logs udpsvd-58cbdc788d-klnwp
     listening on [::]:9999 ...
     connect to [::ffff:172.17.0.10]:9999 from [::ffff:172.17.0.1]:40655 ([::ffff:172.17.0.1]:40655)

## Cleanup

     $ kubectl delete service udpsvd
     $ kubectl delete deployment udpsvd
     $ kubectl delete deployment shell
