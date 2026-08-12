Provisioners let Terraform run scripts or copy files as part of a resource's create/destroy lifecycle. They're the escape hatch for configuring things that have no native provider-managed way to be set up — and HashiCorp is explicit that they should be your last resort, not your default tool. Let's cover the full picture, including why that caution exists.

1. Why provisioners are different from everything else you've learned

Every other Terraform concept so far (resources, data sources, modules) works through the provider's structured schema — Terraform core knows exactly what changed and can compute a clean diff. Provisioners break that model: they're imperative scripts bolted onto a resource's lifecycle, invisible to plan's diff, and their success/failure is opaque to Terraform beyond "exit code 0 or not."

1. local-exec — runs a command on your machine

Executes wherever terraform apply is running (your laptop, or your CI runner) — not on the resource being created.

hcl
resource "aws_instance" "web" {
  ami           = "ami-0abcdef1234567890"
  instance_type = "t3.micro"

  provisioner "local-exec" {
    command = "echo ${self.public_ip} >> inventory.txt"
  }
}

self.<attribute> refers to the resource the provisioner is attached to — the only place you'll see that syntax in Terraform. Common real uses: registering a newly created IP in an external inventory file, triggering a webhook, running a local script that needs the resource's outputs.

hcl
provisioner "local-exec" {
  command     = "python3 notify.py --instance ${self.id}"
  working_dir = "${path.module}/scripts"
  environment = {
    STAGE = "prod"
  }
}
2. file — copies a file or directory to the resource

Requires a connection block since it needs a way to reach the remote machine (SSH for Linux, WinRM for Windows).

hcl
resource "aws_instance" "web" {
  ami           = "ami-0abcdef1234567890"
  instance_type = "t3.micro"
  key_name      = "my-keypair"

  provisioner "file" {
    source      = "app-config/nginx.conf"
    destination = "/tmp/nginx.conf"
  }

  connection {
    type        = "ssh"
    user        = "ubuntu"
    private_key = file("~/.ssh/id_rsa")
    host        = self.public_ip
  }
}

source can be a single file or a directory (copied recursively); destination is the path on the remote machine. Note the file provisioner alone doesn't do anything with the copied file — it just gets it there. You almost always pair it with remote-exec to actually apply/use it.

3. remote-exec — runs commands on the resource itself

Also needs a connection block, and executes inline or scripted commands over that connection.

hcl
resource "aws_instance" "web" {
  ami           = "ami-0abcdef1234567890"
  instance_type = "t3.micro"
  key_name      = "my-keypair"

  provisioner "remote-exec" {
    inline = [
      "sudo mv /tmp/nginx.conf /etc/nginx/nginx.conf",
      "sudo systemctl restart nginx",
    ]
  }

  connection {
    type        = "ssh"
    user        = "ubuntu"
    private_key = file("~/.ssh/id_rsa")
    host        = self.public_ip
  }
}

Alternative form pointing at an existing script instead of inline commands:

hcl
provisioner "remote-exec" {
  script = "scripts/bootstrap.sh"
}

The connection block can be declared once at the resource level (shared by every provisioner on that resource) rather than repeated per-provisioner, as shown combined below.

4. The full pattern: file + remote-exec together

This is the most common real combination — push a config file, then run the commands that install/apply it:

hcl
resource "aws_instance" "web" {
  ami           = "ami-0abcdef1234567890"
  instance_type = "t3.micro"
  key_name      = "my-keypair"

  connection {
    type        = "ssh"
    user        = "ubuntu"
    private_key = file("~/.ssh/id_rsa")
    host        = self.public_ip
  }

  provisioner "file" {
    source      = "app-config/nginx.conf"
    destination = "/tmp/nginx.conf"
  }

  provisioner "remote-exec" {
    inline = [
      "sudo mv /tmp/nginx.conf /etc/nginx/nginx.conf",
      "sudo systemctl restart nginx",
    ]
  }
}

Provisioners on the same resource run in the order they're written, and terraform apply blocks on each one completing before moving to the next.

5. Creation-time vs. destroy-time provisioners

By default, every provisioner runs at creation time. Add when = destroy to run something during teardown instead:

hcl
resource "aws_instance" "web" {
  # ...
  provisioner "local-exec" {
    when    = destroy
    command = "echo 'Instance ${self.id} destroyed' >> audit.log"
  }
}

Destroy-time provisioners can only reference self and can't reach other resources' attributes, since by the time they run, the rest of the graph may already be gone. This is a much less common pattern than creation-time provisioners.

6. on_failure — controlling what a failed provisioner does
hcl
provisioner "remote-exec" {
  inline = ["sudo apt-get install -y nginx"]
  on_failure = continue   # default is "fail"
}
fail (default) — mark the apply as failed and (usually) taint the resource so the next apply recreates it.
continue — log the error but keep going, treating the resource as successfully created anyway.
7. null_resource — provisioners without a real resource

Sometimes you want to run a provisioner that isn't tied to creating any actual infrastructure — e.g., a one-time script after a group of resources exists. null_resource (or its modern replacement, terraform_data) exists exactly for this:

hcl
resource "null_resource" "post_deploy" {
  triggers = {
    instance_ids = join(",", aws_instance.web[*].id)
  }

  provisioner "local-exec" {
    command = "python3 register-with-monitoring.py"
  }
}

triggers controls when the provisioner re-runs — since null_resource has no real attributes to diff against, without triggers it would only ever run once, on first creation.

8. Why HashiCorp says "last resort" — concretely
Invisible to plan — a provisioner's script changes don't show up as a diff; you only find out something changed when apply actually runs it.
No idempotency guarantee — Terraform doesn't know if running the script twice is safe; that's entirely on you to ensure.
Partial-failure state is messy — if remote-exec fails partway through a multi-line inline script, the resource is left in an undefined state (some commands ran, some didn't), and by default the resource gets tainted for full recreation.
Network/connectivity coupling — the connection block requires the resource to be reachable right now, over SSH/WinRM, from wherever apply is running — brittle in CI, firewalled environments, or bastion-only setups.
No dry-run — you can't preview what a provisioner will do the way you can preview a resource diff.
9. What to reach for instead
Cloud-init / user_data — nearly every compute resource type supports passing a boot-time script natively (user_data on aws_instance, custom_data on Azure VMs). This runs at boot without SSH, without a connection block, and is visible as a plain attribute diff in plan.
hcl
  resource "aws_instance" "web" {
    ami           = "ami-0abcdef1234567890"
    instance_type = "t3.micro"
    user_data     = file("bootstrap.sh")
  }
Configuration management tools (Ansible, Chef, Salt) — purpose-built for idempotent, ongoing machine configuration; run as a separate step after terraform apply, not bolted into it.
Baked images (Packer-built AMIs/images with software pre-installed) — push complexity to build time instead of deploy time, so apply just launches an already-correct image.
Provider-native resources — if what you're trying to configure has an actual API (most cloud services do), there's very often a first-class Terraform resource for it already; reach for provisioners only when genuinely nothing else covers the gap.

Rule of thumb: if you find yourself reaching for remote-exec, first ask whether user_data/cloud-init could do the same job — it usually can, and it keeps your configuration inside Terraform's structured, diffable model instead of stepping outside it.