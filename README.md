Spinnaker installation using Minikube and Minio
---------------------------------------------
docker run\
    --name halyard -d \
    -v ${PWD}/hal:/home/spinnaker/.hal \
    -v ${PWD}/kube:/home/spinnaker/.kube \
    -e KUBECONFIG=/home/spinnaker/.kube/config \
    gcr.io/spinnaker-marketplace/halyard:1.28.0

---------------------------------------------
Create a Spinnaker Kubernetes Service account.

CONTEXT=$(kubectl config current-context)

-# This service account uses the ClusterAdmin role -- this is not necessary,
-# more restrictive roles can by applied.
kubectl apply --context $CONTEXT \
    -f https://spinnaker.io/downloads/kubernetes/service-account.yml

TOKEN=$(kubectl get secret --context $CONTEXT \
   $(kubectl get serviceaccount spinnaker-service-account \
       --context $CONTEXT \
       -n spinnaker \
       -o jsonpath='{.secrets[0].name}') \
   -n spinnaker \
   -o jsonpath='{.data.token}' | base64 --decode)

kubectl config set-credentials ${CONTEXT}-token-user --token $TOKEN
kubectl config set-context $CONTEXT --user ${CONTEXT}-token-user

---------------------------------------------
Enable Kubernetes incase of any K8S Deployment
hal config provider kubernetes enable
CONTEXT=$(kubectl config current-context)
hal config provider kubernetes account add minikube_k8s \
    --provider-version v2 \
    --context $CONTEXT
hal config features edit --artifacts true    

---------------------------------------------
Storage service as Minio, we can minio.yaml to deploy and expose service as NodePort  
Given that Minio doesn’t support versioning objects, we need to disable it in Spinnaker. Add the following line to
~/.hal/default/profiles/front50-local.yml
spinnaker.s3.versioning: false


echo $MINIO_SECRET_KEY | hal config storage s3 edit --endpoint $ENDPOINT \
    --access-key-id $MINIO_ACCESS_KEY \
    --secret-access-key
    # will be read on STDIN to avoid polluting your
    # ~/.bash_history with a secret

hal config storage edit --type s3   

-----------------------------------------------
hal config deploy edit --type distributed --account-name minikube_k8s   
hal version list
hal config version edit --version $VERSION
hal deploy apply

-----------------------------------------------

echo minioadmin | hal config storage s3 edit --endpoint http://<ip>:<port> \
    --access-key-id minioadmin \
    --secret-access-key    

----------------------------------------------------------------------------
