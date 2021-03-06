# Running object-detection Project on Azure Cluster
## Initial Setup
  Copy the kubeconfig and save to "kubeconfig" on your local system (using vi editor).
  
    vi kubeconfig  
    export KUBECONFIG=`pwd`/kubeconfig

## Verify the  Cluster
  The Command will disply the azure cluter node to your terminal.
 
    kubectl get nodes 
        NAME                                  STATUS   ROLES   AGE     VERSION
    aks-defaultpool-31705015-vmss000000   Ready    agent   6h40m   v1.19.7


## Verify if the kubeflow is installed.
  ns stands for namespace on your kubernetes cluster.  
  
    kubectl get ns
    kubectl get pods -n kubeflow
    kubectl get pods -n istio-system

## Port forwarding the ingress-gateway service of istio-system namespace
 port forwarding makes possible to access the service from localhost to the cluster. its pre-built service provided by kubernetes. 
   
    kubectl port-forward svc/istio-ingressgateway 7777:80 -n istio-system
        Forwarding from 127.0.0.1:7777 -> 80
    Forwarding from [::1]:7777 -> 80

    
   
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
    
    kubectl get pvc -n kubeflow 
    
        NAME             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
    katib-mysql      Bound    pvc-b4dca223-4097-43ba-bb5d-174b3beb34ba   10Gi       RWO            default        6h18m
    metadata-mysql   Bound    pvc-214e485c-e4f0-4916-940b-b95020e9e73a   10Gi       RWO            default        6h19m
    minio-pv-claim   Bound    pvc-7d0d2776-fa10-4c92-8e9b-f891bc353edc   20Gi       RWO            default        6h18m
    mysql-pv-claim   Bound    pvc-38a05625-95cc-490f-95e9-e362c6968f00   20Gi       RWO            default        6h18m
    pets-pvc         Bound    pvc-52c2f03e-7c64-4a82-8309-a5c0eb886960   20Gi       RWO            default        4h50m


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
 
### creating pet record job
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
  
  
## Monitor the TF-jobs
    kubectl -n kubeflow describe tfjobs tf-training-job
   
   View logs of individual pods
   
    kubectl -n kubeflow get pods
      tf-training-job-master-0                                       1/1     Running            0          147m
      tf-training-job-ps-0                                           1/1     Running            0          147m
      tf-training-job-worker-0                                       1/1     Running            0          147m
      workflow-controller-7dc57f9b8f-vglvl                           1/1     Running            0          6h15m
   
   
    kubectl -n kubeflow logs tf-training-job-master-0 
    
    
## Export the TensorFlow Graph and Serve the model with TF Serving
  Before exporting the graph we first need to identify a checkpoint candidate in the pets-pvc pvc under ${MOUNT_PATH}/train which is where the training job is saving the checkpoints.
  
    kubectl -n kubeflow exec tf-training-job-chief-0 -- ls ${MOUNT_PATH}/train
    
  outputs
  
      checkpoint
    events.out.tfevents.1615272437.tf-training-job-master-0
    events.out.tfevents.1615275113.tf-training-job-master-0
    events.out.tfevents.1615280519.tf-training-job-master-0
    graph.pbtxt
    model.ckpt-0.data-00000-of-00001
    model.ckpt-0.index
    model.ckpt-0.meta
    pipeline.config
    
  Once you have identified the checkpoint next step is to configure the checkpoint in the export-tf-graph-job component and apply it.
  
     CHECKPOINT="${TRAINING_DIR}/model.ckpt-<number>" #replace with your checkpoint number
    INPUT_TYPE="image_tensor"
    EXPORT_OUTPUT_DIR="${MOUNT_PATH}/exported_graphs"

    ks param set export-tf-graph-job mountPath ${MOUNT_PATH}
    ks param set export-tf-graph-job pvc ${PVC}
    ks param set export-tf-graph-job image ${OBJ_DETECTION_IMAGE}
    ks param set export-tf-graph-job pipelineConfigPath ${PIPELINE_CONFIG_PATH}
    ks param set export-tf-graph-job trainedCheckpoint ${CHECKPOINT}
    ks param set export-tf-graph-job outputDir ${EXPORT_OUTPUT_DIR}
    ks param set export-tf-graph-job inputType ${INPUT_TYPE}

    ks apply ${ENV} -c export-tf-graph-job
  
  Once the job is completed a new directory called exported_graphs under /pets_data in the pets-data-claim PCV
will be created containing the model and the frozen graph.

## Serving the Model usinjg the TF-Serving(CPU)

Before serving the model we need to perform q auick hack since the objecy detection export python api does not generate a version folder for saved mdoel. this hack consists on creating a directory and move some files to it. 

    kubectl -n kubeflow exec -it pets-training-master-r1hv-0-i6k7c sh
    mkdir /pets_data/exported_graphs/saved_model/1
    cp /pets_data/exported_graphs/saved_model/* /pets_data/exported_graphs/saved_model/1
  
 ##Configuring the pets-model component in 'ks-app':

     MODEL_COMPONENT=pets-model
    MODEL_PATH=/mnt/exported_graphs/saved_model
    MODEL_STORAGE_TYPE=nfs
    NFS_PVC_NAME=pets-pvc

    ks param set ${MODEL_COMPONENT} modelPath ${MODEL_PATH}
    ks param set ${MODEL_COMPONENT} modelStorageType ${MODEL_STORAGE_TYPE}
    ks param set ${MODEL_COMPONENT} nfsPVC ${NFS_PVC_NAME}

    ks apply ${ENV} -c pets-model
    
  After applying the component see pets model using command and look at the logs 
  
     kubectl -n kubeflow get pods | grep pets-model
     kubectl -n kubeflow logs <pod name>
     
     
  
  
  
    
  
  
  
  
 
    
 
 

    
    
   
  
  
  
    
  






  
  

  
 
    
  
 
 
 

  



