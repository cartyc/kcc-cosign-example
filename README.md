# Signing OCI KCC Artifacts Using `cosign`

## Set env variables

```
PROJECT_ID=<targetprojectid>
KEYRING=<keyring>
KEYNAME=<keyname>
REGION=<region>
REGISTRY_NAME=<registry-name>
CONTAINER_NAME=<container-name>
```

## Enable Required Services
```
gcloud services enable artifactregistry.googleapis.com
gcloud services enable cloudbuild.googleapis.com
gcloud services enable cloudkms.googleapis.com
```

## create keyring 
```
gcloud kms keyrings create $KEYRING \
    --location $REGION
```

## Create Key
```
gcloud kms keys create $KEYNAME \
    --keyring $KEYRING \
    --location northamerica-northeast1 \
    --purpose "asymmetric-signing" \
    --default-algorithm=ec-sign-p256-sha256
```

## Create Artifact Registry

```
gcloud artifacts repositories create ${REGISTRY_NAME} \
    --repository-format=docker \
    --location $REGION
```


## Install Oras and Cosign

```
echo "Install Cosign"
wget https://github.com/sigstore/cosign/releases/download/v1.13.1/cosign-linux-amd64
chmod +x cosign-linux-amd64
mv cosign-linux-amd64 cosign
echo "Install Oras"
curl -LO https://github.com/oras-project/oras/releases/download/v0.16.0/oras_0.16.0_linux_amd64.tar.gz
mkdir -p oras-install/
tar -zxf oras_0.16.0_*.tar.gz -C oras-install/
mv oras-install/oras /usr/local/bin/
rm -rf oras_0.16.0_*.tar.gz oras-install/
```

##  Create some YAML and tar it

```
cat <<EOF > ./gke.yaml
apiVersion: container.cnrm.cloud.google.com/v1beta1
kind: ContainerCluster
metadata:
  name: na-ne1
spec:
  description: An autopilot cluster.
  enableAutopilot: true
  location: $REGION
  releaseChannel:
    channel: REGULAR
---
apiVersion: gkehub.cnrm.cloud.google.com/v1beta1
kind: GKEHubFeatureMembership
metadata:
  name: na-ne1-mesh-membership
spec:
  projectRef:
    external: $PROJECT_ID
  location: global
  membershipRef:
    name: na-ne1-hub
  featureRef:
    name: servicemesh
  mesh:
    management: MANAGEMENT_AUTOMATIC
---
apiVersion: gkehub.cnrm.cloud.google.com/v1beta1
kind: GKEHubMembership
metadata:
  name: na-ne1-hub
spec:
  location: global
  authority:
    issuer: 'https://container.googleapis.com/v1/projects/${PROJECT_ID}/locations/${REGION}/clusters/na-ne1'
  description: na-ne1 Cluster
  endpoint:
    gkeCluster:
      resourceRef:
        name: na-ne1
EOF
```

```
tar -cf ./example.tar gke.yaml
```

## Package and publish YAML
```
SHA=$(oras push ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REGISTRY_NAME}/${CONTAINER_NAME} example.tar | grep Digest: | cut -f2 -d" ")
```

## Set up Cosign
gcloud auth application-default login

```
cosign generate-key-pair --kms gcpkms://projects/${PROJECT_ID}/locations/${REGION}/keyRings/${KEYRING}/cryptoKeys/${KEYNAME}
cosign sign --key gcpkms://projects/${PROJECT_ID}/locations/${REGION}/keyRings/${KEYRING}/cryptoKeys/${KEYNAME} ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REGISTRY_NAME}/${CONTAINER_NAME}@${SHA}
```

## Verify what was created
```
cosign verify --key gcpkms://projects/${PROJECT_ID}/locations/${REGION}/keyRings/${KEYRING}/cryptoKeys/${KEYNAME} ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REGISTRY_NAME}/${CONTAINER_NAME}@${SHA}
```

Voilia you've now signed your KCC infra with cosign!

## Set Up Cosign and Cloud Build

Cloud Build Permissions
Key verifier/signer
key viewer


```
steps:
- name: ubuntu
  script: |
    tar -cf /workspace/example.tar environments
- name: ubuntu
  env:
    - 'PROJECT_ID=$PROJECT_ID'
    - 'COMMIT_SHA=$COMMIT_SHA'
  script: |
    apt update
    apt install wget curl -y
    echo "Install Cosign"
    wget https://github.com/sigstore/cosign/releases/download/v1.13.1/cosign-linux-amd64
    chmod +x cosign-linux-amd64
    mv cosign-linux-amd64 cosign
    echo "Install Oras"
    curl -LO https://github.com/oras-project/oras/releases/download/v0.16.0/oras_0.16.0_linux_amd64.tar.gz
    mkdir -p oras-install/
    tar -zxf oras_0.16.0_*.tar.gz -C oras-install/
    mv oras-install/oras /usr/local/bin/
    rm -rf oras_0.16.0_*.tar.gz oras-install/
    SHA=$(oras push ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REGISTRY_NAME/${CONTAINER_NAME} example.tar | grep Digest: | cut -f2 -d" ")
    echo "SHA"
    echo $SHA
    ./cosign generate-key-pair --kms gcpkms://projects/${PROJECT_ID}/locations/${REGION}/keyRings/${KEYRING}/cryptoKeys/${KEYNAME}
    ./cosign sign --key gcpkms://projects/${PROJECT_ID}/locations/${REGION}/keyRings/${KEYRING}/cryptoKeys/${KEYNAME} ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REGISTRY_NAME/${CONTAINER_NAME@${SHA}
```