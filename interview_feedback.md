Hands-on tasks:

- Write Terraform code to create 5 EC2 instances with unique names and instance types.
- Delete only one specific EC2 instance using Terraform.
- Write a Dockerfile for an application.
-Conditional resource creation in Terraform

Scenario-based questions:

- Difference between Terraform Variables and Locals.
- Difference between "count" and "for_each".
- Using "length()" with "count".
- Recovering from an accidentally deleted Terraform state file.
- Troubleshooting 502 Bad Gateway errors from an AWS Application Load Balancer.
- Debugging database connectivity issues when Kubernetes pods are running.
- Investigating why only 20 out of 40 pods were created after deployment.
- Understanding the Desired State in Kubernetes.
- Why pod-related issues should be investigated in Kubernetes rather than Jenkins.

Kubernetes troubleshooting commands discussed:

- "kubectl get pods"
- "kubectl describe pod"
- "kubectl logs"
- "kubectl get events"
- "kubectl get deployments"
- "kubectl describe deployment"
- "kubectl get svc"
- "kubectl get endpoints"
- "kubectl get nodes"
- "kubectl describe node"
- "kubectl exec -it <pod-name> -- /bin/sh"

My takeaway:

Modern DevOps interviews are moving beyond theoretical questions. Interviewers want to see how you approach production issues, troubleshoot step by step, write Infrastructure as Code, and explain your reasoning while coding. Building strong hands-on skills and understanding real-world scenarios is just as important as knowing the concepts.