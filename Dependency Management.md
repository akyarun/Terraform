# Dependency Management

We touched on the basics of this in your earlier question about depends_on, but "dependency management" as a topic is bigger than just that one meta-argument — it's really about how Terraform builds and walks its entire execution graph. Let's go deep.

## 1. The core idea: Terraform builds a DAG, not a script

Terraform configuration isn't executed top-to-bottom like a script. Every resource, data source, and module call becomes a node in a directed acyclic graph (DAG), and Terraform's engine walks that graph to decide execution order — running independent nodes in parallel wherever possible, and only serializing where an actual dependency exists.

This is worth internalizing: dependency management isn't just about correctness — it's what enables Terraform's parallelism. The security group and IAM role above have no relationship to each other, so Terraform creates them concurrently; only the final instance actually waits. terraform apply -parallelism=N controls the max concurrent operations (default 10) — dependency accuracy is what lets that number actually help you.

## 2. Implicit dependencies — the full picture

You've already seen the core mechanism: an attribute reference (aws_subnet.web.id) creates a graph edge. A few extensions worth knowing:

Module output references also create edges, exactly like resource references:

```
resource "aws_instance" "app" {
  subnet_id = module.network.subnet_id   # depends on everything inside module.network that produces subnet_id
}
```

Splat expressions across count/for_each collections:

```
resource "aws_elb" "web" {
  instances = aws_instance.app[*].id   # depends on ALL instances in the aws_instance.app collection
}
```

Data sources depending on resources (from your earlier data-sources question) work the same way — a data block referencing a resource's attribute creates the same kind of edge, just read instead of write.

## 3. Visualizing the graph yourself

```
terraform graph | dot -Tpng > graph.png
```

terraform graph emits the DOT graph description language; piping through Graphviz's dot renders it as an image. Useful for debugging genuinely confusing dependency chains in larger configs — seeing the actual DAG is often faster than tracing references by eye.

## 4. Explicit dependencies — depends_on, in full

Recap from earlier: use it only when a dependency is real but invisible to attribute references (your IAM eventual-consistency / hardcoded-string examples). The full syntax accepts a list, and works on resources, modules, and (less commonly) data sources:

```
resource "aws_s3_bucket_policy" "logs" {
  # ...
}

resource "aws_cloudtrail" "main" {
  # ...
  depends_on = [aws_s3_bucket_policy.logs]
}
```

On a module block — forces every resource inside the called module to wait, not just one:

```
module "app" {
  source     = "./modules/app"
  depends_on = [module.network]
}
```

This is coarser than resource-level depends_on — it can't express "only these specific resources inside the module depend on this," so it commonly over-serializes more than necessary. Prefer wiring the actual dependency through an output/input pair between modules where you can, and reserve module-level depends_on for cases where that's genuinely not expressible.

On a data source — forces the read to happen only after the listed resource, useful when a data source would otherwise race a resource being created in the same apply:

```
data "aws_subnets" "app" {
  depends_on = [aws_vpc.main]
  filter {
    name   = "vpc-id"
    values = [aws_vpc.main.id]  # this alone usually already creates the edge — depends_on is rarely needed here
  }
}
```

## 5. Destroy order

Destroys run in reverse of the create order the graph implies — dependents get destroyed before their dependencies. In the earlier VPC → subnet → instance chain, terraform destroy tears down the instance first, then the subnet, then the VPC — the opposite direction from apply. This is automatic and follows from the same graph; you don't declare destroy order separately.

## 6. Gotchas worth knowing

* Over-using depends_on kills parallelism you didn't need to sacrifice. Every depends_on is a hard serialization point. If you reach for it reflexively instead of only for genuinely invisible dependencies, you can accidentally chain together resources that had no real reason to wait on each other, and lose the concurrent-execution benefit shown in the diagram above.

* depends_on with count/for_each forces all instances to wait on the entire dependency, not per-instance. If aws_instance.app uses for_each over ten items, and you add depends_on = [aws_iam_role.exec], every one of the ten instances waits for the whole role to exist — which is usually fine, but if you intended a finer-grained per-item dependency, depends_on can't express that; you need an actual attribute reference per item instead.

* depends_on on a data source disables some of Terraform's plan-time visibility for that data source, since it's now guaranteed to be evaluated only after another resource — sometimes deferring its read to apply-time and showing (known after apply) further downstream, similar to the plan-time-visibility gotcha covered in your data-sources question.

* Circular dependencies are a hard error, not silently resolved. If module A's output feeds module B's input, and module B's output feeds module A's input, Terraform can't compute a valid topological order and fails at plan/graph-build time — this has to be broken by restructuring, not fixed with depends_on.

## 7. Practical rule of thumb

1. Reach for a real attribute reference first — always.

2. Only add depends_on when you've confirmed no reference can express the dependency (invisible side effects, eventual consistency, hardcoded values you can't turn into references).

3. Keep depends_on scoped as tightly as possible (resource-level over module-level) to avoid needlessly serializing unrelated work.

4. When a plan or apply order looks wrong or confusingly slow, terraform graph is the actual source of truth — trace it rather than guessing from the .tf files alone.