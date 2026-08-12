# Terraform Outputs 
    — Display Results After Deployment

## What are outputs?

Outputs are how Terraform exposes values from your infrastructure back to you — after apply completes, or to other parts of your Terraform configuration (like other modules). Think of them as the "return values" of your Terraform config.

Without outputs, Terraform creates resources, but you'd have no easy way to see things like the public IP of a server, the ID of a resource, or a generated password — you'd have to go dig through the AWS console or the state file manually.

**Basic Syntax**

    output "<NAME>" {
    value = <EXPRESSION>
    }

**sample example**

    resource "aws_instance" "web" {
    ami           = "ami-0abcdef1234567890"
    instance_type = "t2.micro"
    }

    output "instance_public_ip" {
    value = aws_instance.web.public_ip
    }

**What you see after apply**

    $ terraform apply
    ...
    aws_instance.web: Creation complete after 32s [id=i-0a1b2c3d4e5f]

    Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

    Outputs:

    instance_public_ip = "13.234.56.78"

The value only becomes known after the resource is created — that's exactly why it's useful: you get real, live data back (IP addresses, ARNs, connection strings) that didn't exist before deployment.

### Anatomy of an Output Block

    output "instance_public_ip" {
    description = "The public IP address of the web server"
    value       = aws_instance.web.public_ip
    sensitive   = false
    }

|Argument	        |                   Purpose                                                             |
|------------------|----------------------------------------------------------------------------------------|
|description    	|Documents what the output represents (good practice, shows in docs/tooling)            |
|value	            | The actual expression/value to expose — can reference any resource attribute, variable, or computed expression    |
|sensitive	        |If true, hides the value from CLI output (still stored in state file though — see note below)  |
|depends_on	        |(Rare) explicitly forces this output to wait on a resource, if the dependency isn't inferable automatically    |

## Sensitive Outputs

If an output contains secrets (like a generated password or private key), mark it sensitive = true so it's hidden from normal CLI output:

    resource "aws_db_instance" "database" {
    identifier     = "mydb"
    engine         = "mysql"
    instance_class = "db.t3.micro"
    username       = "admin"
    password       = var.db_password
    allocated_storage = 20
    }

    output "db_password" {
    value     = var.db_password
    sensitive = true
    }

```
$ terraform apply
...
Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

Outputs:

db_password = <sensitive>
```

**Important point**: sensitive = true only hides it from CLI output and logs — the actual value is still stored in plaintext inside the state file. This is why protecting the state file (encryption, restricted access, remote backend) matters just as much as marking outputs sensitive.

To see a sensitive output's actual value deliberately:

```
$ terraform output db_password
"SuperSecret123!"
```
(Running terraform output <name> explicitly reveals it — the sensitive flag only suppresses automatic display during plan/apply.)

## The terraform output Command
    
    You don't have to re-run apply to see outputs again — query them anytime after deployment:

```
$ terraform output

instance_id        = "i-0a1b2c3d4e5f"
instance_public_ip = "13.234.56.78"
vpc_id             = "vpc-0abc123def456789"
```
```
# Show a single specific output
$ terraform output instance_public_ip
"13.234.56.78"
```
```
# Output in raw format (no quotes) — great for scripting
$ terraform output -raw instance_public_ip
13.234.56.78
```
```
# Output as JSON — great for piping into other tools
$ terraform output -json
{
  "instance_id": {
    "sensitive": false,
    "type": "string",
    "value": "i-0a1b2c3d4e5f"
  },
  "instance_public_ip": {
    "sensitive": false,
    "type": "string",
    "value": "13.234.56.78"
  }
}
```

## Best Practices for Outputs

1. Always add description — future you (or teammates) will thank you when reading terraform plan or generated docs.

2. Mark secrets sensitive = true — but remember this only hides CLI display, not state file content.

3. Output only what's useful — don't dump every internal attribute; expose what consumers (humans or other Terraform configs) actually need.
4. Use -raw in scripts — cleaner than parsing quoted JSON-style output manually.

5. Use outputs to document your infrastructure — terraform output after an apply is a fast way to see "what did I just build and how do I connect to it" without digging through the cloud console.