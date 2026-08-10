# Terraform

## Why Infrastructure Automation (Terraform)?

## What is infrastructure?

In a software context, **infrastructure** means all the underlying resources an application needs to run — not the application code itself, but everything it sits on top of:

**Compute**: servers, virtual machines, containers

**Networking**: VPCs, subnets, load balancers, DNS, firewalls

**Storage**: disks, object storage (like S3), databases

**Identity & access**: users, roles, permissions

**Supporting services**: message queues, CDNs, monitoring, secrets managers

Basically, if you deployed an app to the cloud, everything you'd need to click through in AWS/Azure/GCP's console to make that app reachable and functional — that's infrastructure.

## Manual infrastructure vs automated infrastructure

Manual provisioning means a human logs into a cloud console (or SSHs into a server) and creates resources by hand — clicking "Create VM," configuring a security group through a UI, setting up a database instance step by step.

Automated provisioning means you write code that describes the infrastructure you want, and a tool reads that code and creates/updates/destroys the actual resources for you. Terraform is one such tool — you declare "I want a VM with these specs, in this network, with this firewall rule," and Terraform figures out the API calls needed to make that true.

This is often called **Infrastructure as Code (IaC)** : infrastructure defined in files, checked into version control(GIT), just like application code.

## Problems in manual provisioning

a. **Not repeatable / inconsistent** — Doing the same steps twice (e.g., for dev, staging, prod) rarely produces identical environments. Small human differences creep in, leading to the classic "works on staging, breaks in prod" problem.

b. **No version history** — If someone changes a firewall rule or resizes a server manually, there's often no record of what changed, who changed it, or why. Rolling back is guesswork.

c. **Slow and doesn't scale** — Clicking through a console to create one server is fine. Doing it for 50 servers, or recreating an entire environment after a disaster, is painfully slow and error-prone.

d. **Human error** — Manual steps mean typos, forgotten steps, misconfigured settings. A missed checkbox in a security group can cause an outage or a security hole.

e. **No single source of truth** — Nobody can look at one place and know exactly what infrastructure exists. You'd have to log into the console and manually audit everything ("configuration drift" — the real world silently diverging from what people think is deployed).

f. **Hard to collaborate** — Multiple people manually changing the same environment can step on each other's changes, with no merge/review process like code has (pull requests, diffs, approvals).

g. **Poor disaster recovery** — If a manually-built environment gets deleted or corrupted, rebuilding it from memory or scattered documentation is slow and risky.

h. **No testing or preview** — You can't easily "dry run" a manual change to see its blast radius before applying it.

" Automation tools like Terraform solve these by giving you declarative, version-controlled, repeatable, previewable infrastructure — you write the desired end state once, and Terraform handles creating it consistently, every time, anywhere ".

# Introduction to Infrastructure as Code (IaC)

## What is IaC?

Infrastructure as Code (IaC) is the practice of managing and provisioning infrastructure through machine-readable definition files, rather than through manual processes or interactive configuration tools.

Instead of clicking around a cloud console, you write code (in a file) that describes what infrastructure you want — servers, networks, databases, permissions — and a tool reads that file and creates it for you. That code is:

**Version-controlled** (stored in Git, just like application code)

**Reusable** (same code can spin up dev, staging, prod)

**Reviewable** (changes go through pull requests, diffs, approvals)

**Executable** (running the code produces real infrastructure)

Popular IaC tools: Terraform, AWS CloudFormation, Pulumi, Ansible, Azure Resource Manager (ARM) templates.

## Declarative vs Imperative approach

This is one of the most important distinctions in IaC.

### Imperative approach 
    — you specify the exact steps to reach a goal, in order.

**"Create a VM. Then attach a disk. Then install these packages. Then start this service."**

You're telling the system how to do something, step by step. Tools like shell scripts, Ansible (mostly), and Chef tend to lean imperative.

### Declarative approach 
    — you specify the desired end state, and let the tool figure out how to get there.

**"I want 1 VM with this disk, this OS, and this service running."**

You don't say how — you say what. The tool (e.g., Terraform) compares the desired state to the current state and figures out the steps needed to reconcile the difference.

Terraform is declarative.

## Aspect	                    Imperative	                     Declarative
You define	                 Steps to execute	            Desired end state
Order matters?	                Yes, strictly	            Tool figures out order (dependency graph)
Re-running same code	May cause errors/duplicates	        Safe — tool only makes needed changes (idempotent)
Example tools	        Bash scripts, Ansible playbooks	    Terraform, CloudFormation
Mental model	        "Do this, then this, then this"	    "Make it look like this"

### Why declarative matters in practice: 

if you run your Terraform code today, and again next week, Terraform checks the current real-world state against your code and only changes what's different. Run an imperative script twice and you might try to create the same VM twice, or hit an error because a resource already exists — unless someone carefully coded in checks for that.

## Benefits of IaC

1. **Consistency** — Same code = same result, every time, in every environment. No more "it worked when I clicked it manually last time."

2. **Speed** — Spin up entire environments in minutes instead of hours/days of manual clicking.

3. **Version control & audit trail** — Every infrastructure change is a commit. You know who changed what, when, and why — and can roll back if needed.

4. **Collaboration** — Infrastructure changes go through the same review process as code (pull requests, code review) instead of being invisible manual actions.

5. **Reusability** — Write a module once (e.g., "a standard web server setup"), reuse it across projects and teams.

6. **Preview before applying** — Tools like Terraform let you run a "plan" to see exactly what will change before you commit to the change — reducing risk.

7. **Disaster recovery** — If an environment is destroyed, you can rebuild it from code in minutes rather than trying to remember/reconstruct manual steps.

8. **Reduced human error** — No manual clicking means fewer typos, forgotten steps, and misconfigurations.

9. **Documentation as a side effect** — The code itself documents exactly what infrastructure exists — no need for separate, often-outdated wiki pages.

10. **Cost control** — Easier to spin down unused/temporary environments (e.g., a terraform destroy after a demo) since everything is tracked and reproducible.