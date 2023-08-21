# votingApp-Kuber
docker sample voting app Yaml for on-premises Kubernetes
# example-voting-app-kubernetes-v2

This is based on the original [example-voting-app](https://github.com/dockersamples/example-voting-app) from docker-examples(https://github.com/dockersamples)

modified to work on Kubernetes

1) install rke2 master and create some alias in .bashrc
alias kget='kubectl get'
alias kdes='kubectl describe'
alias kapp='kubectl apply -f'
alias kdel='kubectl delete -f'

2) run nexus on docker 
	docker run -d --name nexus -p 8081-8082:8081-8082 -v nexus_data:/nexus-data sonatype/nexus3
	docker volume inspect nexus_data | grep -i mountpoint
	cat /var/lib/docker/volumes/nexus_data/_data/admin.password
	
3) create repository for docker (hosted) on 8082 with annonymous access for download
	1) go to the login page 	http://192.168.56.106:8081/
	2) change the admin password
	3) enable the annonymouse access
	4) create two file blob stroes for helm and docker repositories
	5) create docker hosted repository with http port 8082 in docker blob- allow annonymous download
	6) create helm repository in helm blob 
	7) configure realm - Activate docker bearer token realm and move it to top of list
4) tag the images and push it to nexus

[root@nexus ~]# docker tag dockersamples/examplevotingapp_result nexus.repo:8082/dockersamples/examplevotingapp_result:latest
[root@nexus ~]# docker tag dockersamples/examplevotingapp_vote nexus.repo:8082/dockersamples/examplevotingapp_vote:latest
[root@nexus ~]# docker tag dockersamples/examplevotingapp_worker nexus.repo:8082/dockersamples/examplevotingapp_worker:latest
[root@nexus ~]# docker tag redis:alpine nexus.repo:8082/redis:alpine
[root@nexus ~]# docker tag postgres:15-alpine nexus.repo:8082/postgres:15-alpine

5) configure docker to login to unsecure repository

[root@nexus ~]# cat /etc/docker/daemon.json
{
  "insecure-registries" : ["nexus.repo:8082"]
}

[root@nexus ~]# systemctl restart docker
[root@nexus ~]# docker start nexus
[root@nexus ~]# docker login nexus.repo:8082

6) push the images into the registry
7) define default registry for rke2 on all nodes
[root@docker1 ~]# cat /etc/rancher/rke2/registries.yaml
mirrors:
  docker.io:
    endpoint:
      - "http://nexus.repo:8082"
  nexus.repo:8082:
    endpoint:
      - "http://nexus.repo:8082"

[root@docker1 ~]# systemctl restart rke2-server|rke2-agent

8) create a new context and change the defualt namespace
	kubectl config set-context context-voteApp --cluster default --user default --namespace vote-ns
  	kubectl create ns vote-ns
	kubectl config use-context context-voteApp
9) install all the yaml files from yaml file folder

10) verify the installation 
[root@docker1 votingAppYaml]# kget all  --show-labels
NAME                 READY   STATUS    RESTARTS   AGE     LABELS
pod/postgres-pod     1/1     Running   0          7m8s    app=demo-voting-app,name=postgres-pod
pod/redis-pod        1/1     Running   0          9m46s   app=demo-voting-app,name=redis-pod
pod/result-app-pod   1/1     Running   0          12m     app=demo-voting-app,name=result-app-pod
pod/voting-app-pod   1/1     Running   0          31m     app=demo-voting-app,name=voting-app-pod
pod/worker-app-pod   1/1     Running   0          12m     app=demo-voting-app,name=worker-app-pod

NAME                     TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE     LABELS
service/db               ClusterIP   10.43.63.156   <none>        5432/TCP       86s     app=demo-voting-app,name=db-service
service/redis            ClusterIP   10.43.154.19   <none>        6379/TCP       5m12s   app=demo-voting-app,name=redis-service
service/result-service   NodePort    10.43.5.125    <none>        80:30008/TCP   17s     app=demo-result-app,name=result-service
service/voting-service   NodePort    10.43.250.27   <none>        80:30007/TCP   25m     app=demo-voting-app,name=voting-service

http://node-ip:30007
http://node-ip:30008

