## MaxDiffusion Inference on Google Kubernetes Engine (GKE)

This demo will run through containerizing StableDiffusionXL (SDXL) from [MaxDiffusion](https://github.com/google/maxdiffusion), deploying to a GKE cluster, and performing inference on v4 TPUs.


### Setup environment variables

```
gcloud config set project [PROJECT_ID]
export PROJECT_ID=$(gcloud config get project)
export REGION=[COMPUTE_REGION]
export ZONE=[ZONE]
export SDXL_IMG = [STABLE_DIFFUSION_XL_CONTAINER_IMAGE_NAME]
export VM_NAME = [VIRTUAL_MACHINE_NAME]
export CLUSTER_NAME = [CLUSTER_NAME]
export AR_NAME = [ARTIFACT_REGISTRY_NAME]
```

### Create Prerequisite GCP Resources


For building the container image, create a Compute Engine TPU VM:
```
gcloud compute tpus tpu-vm create maxdiffusion-vm \
--project=${PROJECT_ID} \
--zone=${ZONE} \
--accelerator-type=v4-8 \
--version=tpu-vm-tf-2.16.1-pjrt 
--tags=ssh-tunnel-iap
```

For storing the built image, use an Artifact Registry repo:
```
gcloud artifacts repositories create ${AR_NAME} \
    --repository-format=docker \
    --location=${REGION} \
    --immutable-tags \
    --async
```

Create a standard GKE cluster with a TPU v5 node pool with a 1x1 topology:
```
gcloud container clusters create ${CLUSTER_NAME} \
  --enable-ip-alias \
  --machine-type=n1-standard-8 \
  --workload-pool=${PROJECT_ID}.svc.id.goog \
  --release-channel=rapid \
  --zone=${ZONE}
&& gcloud container node-pools create tpu-nodepool \
  --cluster=${CLUSTER_NAME} \
  --machine-type=ct5lp-hightpu-1t \
  --zone=${ZONE} \
  --num-nodes=1
```

Use gcloud to configure kubectl so that it communicates with the cluster:
```
gcloud container clusters get-credentials maxdiffusion-cluster --location=us-central2-b
```

### Building and Serving the Container Image

Once that is complete, connect to the VM via SSH by navigating to `Compute Engine > TPUs` in the Cloud Console and click SSH in the [VIRTUAL_MACHINE_NAME] VM. Once logged in, run the following to pull the correct MaxDiffusion branch (or Master if the branch has since been merged), navigate to the dockerfile:

```
git clone https://github.com/google/maxdiffusion.git \
&& cd maxdiffusion \
&& git checkout inf_mlperf \
&& cd docker/sdxl_inference/dockerfile
```

Next, build the container image with [the tag name that Artifact Registry expects](https://cloud.google.com/artifact-registry/docs/docker/pushing-and-pulling#tag) and push it to the new repo:
```
docker build -t us-central2-docker.pkg.dev/maxdiffusion-project/maxdiffusion-repo/maxdiffusion-image . \
&& docker push us-central2-docker.pkg.dev/maxdiffusion-project/maxdiffusion-repo/maxdiffusion-image
```

### Deploying MaxDiffusion on GKE

```
kubectl apply -f maxdiffusion_pipeline.yaml
```

Inspect all pods in the namespace default, there should be one pod with the prefix maxdiffusion-server-v5-1x1:
```
kubectl get pods --namespace default
NAME                                            READY   STATUS    RESTARTS   AGE
maxdiffusion-server-v5-1x1-795b4d65f7-5tt57   1/1     Running   0          1m
```

### Making Inference

The container exposes two HTTP endpoints on port 8080, /health for health checking and /predict which accepts a JSON payload with your text query. First expose the model server endpoints via port forwarding:

```
kubectl port-forward deployment/maxdiffusion-server-v4-2x2x1 8080:8080
```

Next, in a different terminal, you can make your request via cURL:
```
curl --header "Content-Type: application/json" -X POST -d '{"instances":[{"prompt":["a dog walking a cat"],"query_id":["1"]}]}' http://localhost:8080/predict
```

This should result in a very large JSON response mainly made up of the image encoded as a base64 string under the images field. Typically the decoding process would be automated but for this demo we can decode the image by hand using an online tool such as Code Beautify, simply paste the string in to see the decoded image:
![image](maxdiffusion_result.jpeg)