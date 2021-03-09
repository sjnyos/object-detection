# Running object-detection Project on Azure Cluster
## Initial Setup
  Copy the kubeconfig and save to "kubeconfig" on your local system (using vi editor).
  
    vi kubeconfig  
    export KUBECONFIG=`pwd`/kubeconfig

## Verify the  Cluster
  The Command will disply the azure cluter node to your terminal.
 
    kubectl get nodes 

## Verify if the kubeflow is installed.
  ns stands for namespace on your kubernetes cluster.  
  
    kubectl get ns
    kubectl get pods -n kubeflow
    kubectl get pods -n istio-system

## Port forwarding the ingress-gateway service of istio-system namespace
 port forwarding makes possible to access the service from localhost to the cluster. its pre-built service provided by kubernetes. 
   
    kubectl port-forward svc/istio-ingressgateway 7777:80 -n istio-system
   
 ## Go to your web Browser localhost:7777 for kubeflow dash borad.
     
    http://localhost:7777 
 
 
 ## Requirements
  [] Ksonnet CLI: ks
  [] Kubernetes cluster
  
## Creating the KS project
    APP_NAME=ks-app
    ks init ${APP_NAME} --api-spec=version:v1.7.0
    cd ks-app
    ENV=default
    ks env add ${ENV} --context=`kubectl config current-context`
    ks env set ${ENV} --namespace kubeflow

## Creating the PVC
    ks param set pets-pvc accessMode "ReadWriteOnce"
    ks param set pets-pvc storage "20Gi"
    ks apply ${ENV} -c pets-pvc
    
## Preparing the Training Data 
    # Configure and apply the get-data-job component this component will download the dataset,
    # annotations, the model we will use for the fine tune checkpoint, and
    # the pipeline configuration file
    PVC="pets-pvc"
    MOUNT_PATH="/pets_data"
    DATASET_URL="http://www.robots.ox.ac.uk/~vgg/data/pets/data/images.tar.gz"
    ANNOTATIONS_URL="http://www.robots.ox.ac.uk/~vgg/data/pets/data/annotations.tar.gz"
    MODEL_URL="http://download.tensorflow.org/models/object_detection/faster_rcnn_resnet101_coco_2018_01_28.tar.gz"
    PIPELINE_CONFIG_URL="https://raw.githubusercontent.com/kubeflow/examples/master/object_detection/conf/faster_rcnn_resnet101_pets.config"

    ks param set get-data-job mounthPath ${MOUNT_PATH}
    ks param set get-data-job pvc ${PVC}
    ks param set get-data-job urlData ${DATASET_URL}
    ks param set get-data-job urlAnnotations ${ANNOTATIONS_URL}
    ks param set get-data-job urlModel ${MODEL_URL}
    ks param set get-data-job urlPipelineConfig ${PIPELINE_CONFIG_URL}

    ks apply ${ENV} -c get-data-job
 
 ### Decompress data job
    ANNOTATIONS_PATH="${MOUNT_PATH}/annotations.tar.gz"
    DATASET_PATH="${MOUNT_PATH}/images.tar.gz"
    PRE_TRAINED_MODEL_PATH="${MOUNT_PATH}/faster_rcnn_resnet101_coco_2018_01_28.tar.gz"

    ks param set decompress-data-job mountPath ${MOUNT_PATH}
    ks param set decompress-data-job pvc ${PVC}
    ks param set decompress-data-job pathToAnnotations ${ANNOTATIONS_PATH}
    ks param set decompress-data-job pathToDataset ${DATASET_PATH}
    ks param set decompress-data-job pathToModel ${PRE_TRAINED_MODEL_PATH}

    ks apply ${ENV} -c decompress-data-job
 
### pet record job
    OBJ_DETECTION_IMAGE="lcastell/pets_object_detection"
    DATA_DIR_PATH="${MOUNT_PATH}"
    OUTPUT_DIR_PATH="${MOUNT_PATH}"

    ks param set create-pet-record-job image ${OBJ_DETECTION_IMAGE}
    ks param set create-pet-record-job dataDirPath ${DATA_DIR_PATH}
    ks param set create-pet-record-job outputDirPath ${OUTPUT_DIR_PATH}
    ks param set create-pet-record-job mountPath ${MOUNT_PATH}
    ks param set create-pet-record-job pvc ${PVC}

    ks apply ${ENV} -c create-pet-record-job
    
 ## Creating training TF-Job Deployment and launing it.
 
     # from the ks-app directory

    PIPELINE_CONFIG_PATH="${MOUNT_PATH}/faster_rcnn_resnet101_pets.config"
    TRAINING_DIR="${MOUNT_PATH}/train"

    ks param set tf-training-job image ${OBJ_DETECTION_IMAGE}
    ks param set tf-training-job mountPath ${MOUNT_PATH}
    ks param set tf-training-job pvc ${PVC}
    ks param set tf-training-job numPs 1
    ks param set tf-training-job numWorkers 1
    ks param set tf-training-job pipelineConfigPath ${PIPELINE_CONFIG_PATH}
    ks param set tf-training-job trainDir ${TRAINING_DIR}

    ks apply ${ENV} -c tf-training-job
    
NOTE: The default TFJob api verison in the component is kubeflow.org/v1beta1. You can override the default version by setting the tfjobApiVersion param in the ksonnet app

    NEW_VERSION= kubeflow.org/v1
    ks param set tf-training-job tfjobApiVersion ${NEW_VERSION}
    
     ks apply ${ENV} -c tf-training-job
  
  
    
  






  
  

  
 
    
  
 
 
 

  



