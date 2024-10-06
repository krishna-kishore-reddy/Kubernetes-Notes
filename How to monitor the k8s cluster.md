How to add prometheus and grafana to our k8's cluster for real time monitoring
=

root@kubemaster ~ # helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
"prometheus-community" has been added to your repositories

root@kubemaster ~ # helm repo update

Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "ingress-nginx" chart repository
...Successfully got an update from the "prometheus-community" chart repository
Update Complete. ⎈Happy Helming!⎈

root@kubemaster ~ # helm install prometheus prometheus-community/kube-prometheus-stack

NAME: prometheus
LAST DEPLOYED: Sun Oct  6 04:38:56 2024
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
kube-prometheus-stack has been installed. Check its status by running:
  kubectl --namespace default get pods -l "release=prometheus"

Visit https://github.com/prometheus-operator/kube-prometheus for instructions on how to create & configure Alertmanager and Prometheus instances using the Operator.
root@kubemaster ~ # 

root@kubemaster ~ # kubectl get pods
NAME                                                     READY   STATUS            RESTARTS         AGE
alertmanager-prometheus-kube-prometheus-alertmanager-0   0/2     PodInitializing   0                13s
bridge-64d4dc4446-nppzz                                  1/1     Running           17 (9m39s ago)   17d
ingress-nginx-controller-c8f499cfc-qctsm                 1/1     Running           74 (9m39s ago)   117d
prometheus-grafana-7b6d94ffb4-jjct6                      2/3     Running           0                21s
prometheus-kube-prometheus-operator-846fdd5848-kpsb6     1/1     Running           0                21s
prometheus-kube-state-metrics-5b787f976b-4bz4g           1/1     Running           0                21s
prometheus-prometheus-kube-prometheus-prometheus-0       0/2     PodInitializing   0                13s
prometheus-prometheus-node-exporter-88hwx                1/1     Running           0                21s
prometheus-prometheus-node-exporter-9m5hf                1/1     Running           0                21s
prometheus-prometheus-node-exporter-lv8z5                1/1     Running           0                21s
tableau-bridge-6fc4dc7bb6-54d5b 


root@kubemaster ~ # helm search repo prometheus-community
NAME                                                    CHART VERSION   APP VERSION     DESCRIPTION                                       
prometheus-community/alertmanager                       1.13.0          v0.27.0         The Alertmanager handles alerts sent by client ...
prometheus-community/alertmanager-snmp-notifier         0.3.0           v1.5.0          The SNMP Notifier handles alerts coming from Pr...
prometheus-community/jiralert                           1.7.1           v1.3.0          A Helm chart for Kubernetes to install jiralert   
prometheus-community/kube-prometheus-stack              65.0.0          v0.77.1         kube-prometheus-stack collects Kubernetes manif...
prometheus-community/kube-state-metrics                 5.25.1          2.13.0          Install kube-state-metrics to generate and expo...
prometheus-community/prom-label-proxy                   0.10.0          v0.11.0         A proxy that enforces a given label in a given ...
prometheus-community/prometheus                         25.27.0         v2.54.1         Prometheus is a monitoring system and time seri...
prometheus-community/prometheus-adapter                 4.11.


root@kubemaster ~ # kubectl get svc
NAME                                      TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
alertmanager-operated                     ClusterIP      None             <none>        9093/TCP,9094/TCP,9094/UDP   4m34s
ingress-nginx-controller                  LoadBalancer   10.98.76.81      <pending>     80:31716/TCP,443:30484/TCP   156d
ingress-nginx-controller-admission        ClusterIP      10.110.180.86    <none>        443/TCP                      156d
jenkins-svc                               ClusterIP      10.97.14.163     <none>        8080/TCP                     5d15h
kubernetes                                ClusterIP      10.96.0.1        <none>        443/TCP                      164d
nginx-svc                                 ClusterIP      10.110.185.255   <none>        80/TCP                       5d15h
prometheus-grafana                        ClusterIP      10.110.228.200   <none>        80/TCP                       4m42s
prometheus-kube-prometheus-alertmanager   ClusterIP      10.108.49.24     <none>        9093/TCP,8080/TCP            4m42s
prometheus-kube-prometheus-operator       ClusterIP      10.106.192.110   <none>        443/TCP                      4m42s
prometheus-kube-prometheus-prometheus     ClusterIP      10.100.59.207    <none>        9090/TCP,8080/TCP            4m42s
prometheus-kube-state-metrics             ClusterIP      10.105.28.146    <none>        8080/TCP                     4m42s
prometheus-operated                       ClusterIP      None             <none>        9090/TCP                     4m34s
prometheus-prometheus-node-exporter       ClusterIP      10.107.222.75    <none>        9100/TCP                     4m42s


root@kubemaster ~ # kubectl edit svc prometheus-grafana    #eidt the SVC to change the clusterip to nodeport to access the grafana dashboard 
service/prometheus-grafana edited
