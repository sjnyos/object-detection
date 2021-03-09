# Running object-detection Project on Azure Cluster
## Initial Setup
Copy the kubeconfig and save to "kubeconfig" on your local system (using vi editor)

  commands on terminal:
  
    vi kubeconfig  
    export KUBECONFIG=`pwd`/kubeconfig

Verify the Azure cluster
 
    kubectl get nodes 

Verify if the kubeflow is installed.
  
    kubectl get ns
    kubectl get pods -n kubeflow
    kubectl get pods -n istio-system

Port forwarding the ingress-gateway service of istio-system namespace
   ### runing on port 7777
     kubectl port-forward svc/istio-ingressgateway 7777:80 -n istio-system
   
 Go to your web Browser localhost:7777 for kubeflow dash borad.
     
    http://localhost:7777 
    
 
    
  
 
 
 

  



