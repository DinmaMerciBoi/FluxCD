# FluxCD
Configuring FluxCD

# 1. Install the Flux CLI
With Bash for macOS and Linux:

curl -s https://fluxcd.io/install.sh | sudo bash

# 2. Export your credentials 
Export your GitHub personal access token and username:

export GITHUB_TOKEN=<your-token>
export GITHUB_USER=<your-username>

# 3. Check your Kubernetes cluster

flux check --pre

The output is similar to:

► checking prerequisites
✔ kubernetes 1.22.2 >=1.20.6
✔ prerequisites checks passed

# 4. Install Flux onto your cluster
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=fleet-infra \
  --branch=main \
  --path=./clusters/my-cluster \
  --personal

The bootstrap command above does the following:
*Creates a git repository fleet-infra on your GitHub account.
*Adds Flux component manifests to the repository.
*Deploys Flux Components to your Kubernetes Cluster.
*Configures Flux components to track the path /clusters/my-cluster/ in the repository.

# 5. Clone the git repository
Clone the fleet-infra repository to your local machine:

git clone https://github.com/$GITHUB_USER/fleet-infra

Or

git clone https://$GITHUB_USER:$GITHUB_TOKEN@github.com/$GITHUB_USER/fleet-infra.git

cd fleet-infra

# 6. Add custom (or podinfo, as example) repository to Flux
URL(podinfo)=https://github.com/DinmaMerciBoi/podinfo

*Create a GitRepository manifest pointing to podinfo repository’s master branch:

flux create source git podinfo \
  --url=https://github.com/stefanprodan/podinfo \
  --branch=master \
  --interval=30s \
  --export > ./clusters/my-cluster/podinfo-source.yaml

*Commit and push the podinfo-source.yaml file to the fleet-infra repository:

git add -A && git commit -m "Add podinfo GitRepository"
git push

# 6. Deploy podinfo application

*Use the flux create command to create a Kustomization that applies the podinfo deployment.

flux create kustomization podinfo \
  --target-namespace=default \
  --source=podinfo \
  --path="./kustomize" \
  --prune=true \
  --interval=5m \
  --export > ./clusters/my-cluster/podinfo-kustomization.yaml

*Commit and push the Kustomization manifest to the repository:

git add -A && git commit -m "Add podinfo Kustomization"
git push

# 7. Watch Flux sync the application

*Use the flux get command to watch the podinfo app.

flux get kustomizations --watch

*Check podinfo has been deployed on your cluster:

kubectl -n default get deployments,services

Changes made to the podinfo Kubernetes manifests in the master branch are reflected in your cluster.
When a Kubernetes manifest is removed from the podinfo repository, Flux removes it from your cluster. When you delete a Kustomization from the fleet-infra repository, Flux removes all Kubernetes objects previously applied from that Kustomization.
When you alter the podinfo deployment using kubectl edit, the changes are reverted to match the state described in Git.

# 8. Suspend updates
Suspending updates to a kustomization allows you to directly edit objects applied from a kustomization, without your changes being reverted by the state in Git.
To suspend updates for a kustomization, run the command 

flux suspend kustomization name

To resume updates run the command
  
flux resume kustomization name

# 9. Customize podinfo deployment
To customize a deployment from a repository you don’t control, you can use Flux in-line patches. The following example shows how to use in-line patches to change the podinfo deployment.

Add the following to the field spec of your podinfo-kustomization.yaml file:

patches:
    - patch: |-
        apiVersion: autoscaling/v2beta2
        kind: HorizontalPodAutoscaler
        metadata:
          name: podinfo
        spec:
          minReplicas: 3             
      target:
        name: podinfo
        kind: HorizontalPodAutoscaler

*Commit and push the podinfo-kustomization.yaml changes:

git add -A && git commit -m "Increase podinfo minimum replicas"
git push

After the synchronization finishes, running kubectl get pods should display 3 pods.

# For further reading,
See https://fluxcd.io/flux/get-started/
