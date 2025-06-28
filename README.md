# GKE Deployment Instructions

## Prerequisites

1. **Install required tools:**
   ```bash
   # Install Google Cloud SDK
   curl https://sdk.cloud.google.com | bash
   exec -l $SHELL
   
   # Install Terraform
   wget https://releases.hashicorp.com/terraform/1.0.0/terraform_1.0.0_linux_amd64.zip
   unzip terraform_1.0.0_linux_amd64.zip
   sudo mv terraform /usr/local/bin/
   
   # Install kubectl
   curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
   chmod +x kubectl
   sudo mv kubectl /usr/local/bin/
   ```

2. **Authenticate with Google Cloud:**
   ```bash
   gcloud auth login
   gcloud config set project YOUR_PROJECT_ID
   gcloud auth application-default login
   ```

## Step 1: Deploy Infrastructure with Terraform

1. **Create terraform.tfvars file:**
   ```bash
   cat > terraform.tfvars << EOF
   project_id = "your-gcp-project-id"
   region     = "us-central1"
   zone       = "us-central1-a"
   cluster_name = "apache-cluster"
   EOF
   ```

2. **Initialize and deploy Terraform:**
   ```bash
   terraform init
   terraform plan
   terraform apply
   ```

3. **Get cluster credentials:**
   ```bash
   gcloud container clusters get-credentials apache-cluster --zone us-central1-a --project YOUR_PROJECT_ID
   ```

## Step 2: Deploy Apache Application

1. **Apply the Kubernetes manifests:**
   ```bash
   # Deploy Apache application
   kubectl apply -f apache-deployment.yaml

   kubectl apply -f apache-configmap.yaml
   kubectl apply -f apache-ingress.yaml
   kubectl apply -f apache-service.yaml

   
   # Check deployment status
   kubectl get deployments
   kubectl get pods -o wide
   ```

2. **Apply Network Policies:**
   ```bash
   kubectl apply -f network-policies.yaml

   kubectl apply -f network-policy-ingress.yaml
   kubectl apply -f network-policy-egress.yaml
   kubectl apply -f deny-all-default-policy.yaml
   kubectl apply -f master-node-policy.yaml
   kubectl apply -f worker-node-policy.yaml
   




   
   # Verify network policies
   kubectl get networkpolicies
   ```

## Step 3: Access Your Application

1. **Get the LoadBalancer IP:**
   ```bash
   kubectl get service apache-service
   
   # Wait for EXTERNAL-IP to be assigned (may take a few minutes)
   kubectl get service apache-service --watch
   ```

2. **Access your application:**
   ```bash
   # Get the external IP
   EXTERNAL_IP=$(kubectl get service apache-service -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
   echo "Apache is accessible at: http://$EXTERNAL_IP"
   
   # Test the application
   curl http://$EXTERNAL_IP
   ```

## Step 4: Verify Node Roles

1. **Check node labels:**
   ```bash
   kubectl get nodes --show-labels
   
   # You should see nodes with role=master and role=worker labels
   ```

2. **Verify pod placement:**
   ```bash
   kubectl get pods -o wide
   
   # Apache pods should be running on worker nodes
   ```

## Step 5: Set up Ingress (Optional)

1. **Reserve a static IP:**
   ```bash
   gcloud compute addresses create apache-ip --global
   ```

2. **Create managed SSL certificate:**
   ```bash
   cat > managed-cert.yaml << EOF
   apiVersion: networking.gke.io/v1
   kind: ManagedCertificate
   metadata:
     name: apache-ssl-cert
   spec:
     domains:
       - apache.example.com  # Replace with your domain
   EOF
   
   kubectl apply -f managed-cert.yaml
   ```

3. **Apply the ingress:**
   ```bash
   kubectl apply -f apache-ingress.yaml
   
   # Check ingress status
   kubectl get ingress apache-ingress
   ```

## Monitoring and Troubleshooting

1. **Check pod logs:**
   ```bash
   kubectl logs -l app=apache
   ```

2. **Describe services:**
   ```bash
   kubectl describe service apache-service
   ```

3. **Check network policies:**
   ```bash
   kubectl describe networkpolicy
   ```

4. **Monitor cluster:**
   ```bash
   kubectl top nodes
   kubectl top pods
   ```

## Scaling

1. **Scale the deployment:**
   ```bash
   kubectl scale deployment apache-deployment --replicas=4
   ```

2. **Enable horizontal pod autoscaling:**
   ```bash
   kubectl autoscale deployment apache-deployment --cpu-percent=50 --min=2 --max=10
   ```

## Cleanup

1. **Delete Kubernetes resources:**
   ```bash
   kubectl delete -f apache-deployment.yaml
   kubectl delete -f network-policies.yaml
   kubectl delete ingress apache-ingress
   ```

2. **Destroy Terraform infrastructure:**
   ```bash
   terraform destroy
   ```

## Important Notes

- **Node Roles**: In GKE, the concept of "master" and "worker" nodes is handled differently than in self-managed Kubernetes. GKE manages the control plane for you. The node pools created are labeled with `role=master` and `role=worker` for organization purposes.

- **Network Policies**: The network policies provide basic security. Adjust the CIDR blocks and rules according to your security requirements.

- **LoadBalancer**: The LoadBalancer service will create a Google Cloud Load Balancer automatically. This may incur additional costs.

- **SSL/TLS**: For production use, configure proper SSL certificates and use HTTPS.

- **Monitoring**: Consider enabling Google Cloud Monitoring and Logging for better observability.

## Security Best Practices

1. Use least privilege access
2. Regularly update your container images
3. Implement proper network policies
4. Use Google Cloud Security Command Center
5. Enable audit logging
6. Use Workload Identity for secure access to Google Cloud services
