
# terraform-null-label


Terraform module designed to generate consistent names and tags for resources. Use `terraform-null-label` to implement a strict naming convention.

A label follows the following convention: `{namespace}-{environment}-{stage}-{name}-{attributes}`. The delimiter (e.g. `-`) is interchangeable.
The label items are all optional. So if you prefer the term `stage` to `environment` you can exclude environment and the label `id` will look like `{namespace}-{stage}-{name}-{attributes}`.
If attributes are excluded but `stage` and `environment` are included, `id` will look like `{namespace}-{environment}-{stage}-{name}`

It's recommended to use one `terraform-null-label` module for every unique resource of a given resource type.
For example, if you have 10 instances, there should be 10 different labels.
However, if you have multiple different kinds of resources (e.g. instances, security groups, file systems, and elastic ips), then they can all share the same label assuming they are logically related.

All [Cloud Posse modules](https://github.com/cloudposse?utf8=%E2%9C%93&q=terraform-&type=&language=) use this module to ensure resources can be instantiated multiple times within an account and without conflict.

**NOTE:** The `null` refers to the primary Terraform [provider](https://www.terraform.io/docs/providers/null/index.html) used in this module.

Releases of this module from `0.12.0` onward support `HCL2` and only work with Terraform 0.12 or newer.  Releases prior to this are compatible with earlier versions of terraform like Terraform 0.11.


---

This project is part of our comprehensive ["SweetOps"](https://cpco.io/sweetops) approach towards DevOps. 

![Terraform Open Source Modules](https://docs.cloudposse.com/images/terraform-open-source-modules.svg)



It's 100% Open Source and licensed under the [APACHE2](LICENSE).







We literally have [*hundreds of terraform modules*][terraform_modules] that are Open Source and well-maintained. Check them out! 







## Usage


**IMPORTANT:** The `master` branch is used in `source` just as an example. In your code, do not pin to `master` because there may be breaking changes between releases.
Instead pin to the release tag (e.g. `?ref=tags/x.y.z`) of one of our [latest releases](https://github.com/cloudposse/terraform-null-label/releases).


### Simple Example

```hcl
module "eg_prod_bastion_label" {
  source     = "git::https://github.com/cloudposse/terraform-null-label.git?ref=master"
  namespace  = "eg"
  stage      = "prod"
  name       = "bastion"
  attributes = ["public"]
  delimiter  = "-"

  tags = {
    "BusinessUnit" = "XYZ",
    "Snapshot"     = "true"
  }
}
```

This will create an `id` with the value of `eg-prod-bastion-public` because when generating `id`, the default order is `namespace`, `environment`, `stage`,  `name`, `attributes`
(you can override it by using the `label_order` variable, see [Advanced Example 3](#advanced-example-3)).

Now reference the label when creating an instance:

```hcl
resource "aws_instance" "eg_prod_bastion_public" {
  instance_type = "t1.micro"
  tags          = module.eg_prod_bastion_label.tags
}
```

Or define a security group:

```hcl
resource "aws_security_group" "eg_prod_bastion_public" {
  vpc_id = var.vpc_id
  name   = module.eg_prod_bastion_label.id
  tags   = module.eg_prod_bastion_label.tags
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```


### Advanced Example

Here is a more complex example with two instances using two different labels. Note how efficiently the tags are defined for both the instance and the security group.

```hcl
module "eg_prod_bastion_abc_label" {
  source     = "git::https://github.com/cloudposse/terraform-null-label.git?ref=master"
  namespace  = "eg"
  stage      = "prod"
  name       = "bastion"
  attributes = ["abc"]
  delimiter  = "-"

  tags = {
    "BusinessUnit" = "XYZ",
    "Snapshot"     = "true"
  }
}

resource "aws_security_group" "eg_prod_bastion_abc" {
  name = module.eg_prod_bastion_abc_label.id
  tags = module.eg_prod_bastion_abc_label.tags
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_instance" "eg_prod_bastion_abc" {
   instance_type          = "t1.micro"
   tags                   = module.eg_prod_bastion_abc_label.tags
   vpc_security_group_ids = [aws_security_group.eg_prod_bastion_abc.id]
}

module "eg_prod_bastion_xyz_label" {
  source     = "git::https://github.com/cloudposse/terraform-null-label.git?ref=master"
  namespace  = "eg"
  stage      = "prod"
  name       = "bastion"
  attributes = ["xyz"]
  delimiter  = "-"

  tags = {
    "BusinessUnit" = "XYZ",
    "Snapshot"     = "true"
  }
}

resource "aws_security_group" "eg_prod_bastion_xyz" {
  name = module.eg_prod_bastion_xyz_label.id
  tags = module.eg_prod_bastion_xyz_label.tags
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_instance" "eg_prod_bastion_xyz" {
   instance_type          = "t1.micro"
   tags                   = module.eg_prod_bastion_xyz_label.tags
   vpc_security_group_ids = [aws_security_group.eg_prod_bastion_xyz.id]
}
```

### Advanced Example 2

Here is a more complex example with an autoscaling group that has a different tagging schema than other resources and requires its tags to be in this format, which this module can generate:

```hcl
tags = [
    {
        key = Name,
        propagate_at_launch = 1,
        value = namespace-stage-name
    },
    {
        key = Namespace,
        propagate_at_launch = 1,
        value = namespace
    },
    {
        key = Stage,
        propagate_at_launch = 1,
        value = stage
    }
]
```

Autoscaling group using propagating tagging below (full example: [autoscalinggroup](examples/autoscalinggroup/main.tf))

```hcl
################################
# terraform-null-label example #
################################
module "label" {
  source    = "../../"
  namespace = "cp"
  stage     = "prod"
  name      = "app"

  tags = {
    BusinessUnit = "Finance"
    ManagedBy    = "Terraform"
  }

  additional_tag_map = {
    propagate_at_launch = "true"
  }
}

#######################
# Launch template     #
#######################
resource "aws_launch_template" "default" {
  # terraform-null-label example used here: Set template name prefix
  name_prefix                           = "${module.label.id}-"
  image_id                              = data.aws_ami.amazon_linux.id
  instance_type                         = "t2.micro"
  instance_initiated_shutdown_behavior  = "terminate"

  vpc_security_group_ids                = [data.aws_security_group.default.id]

  monitoring {
    enabled                             = false
  }
  # terraform-null-label example used here: Set tags on volumes
  tag_specifications {
    resource_type                       = "volume"
    tags                                = module.label.tags
  }
}

######################
# Autoscaling group  #
######################
resource "aws_autoscaling_group" "default" {
  # terraform-null-label example used here: Set ASG name prefix
  name_prefix                           = "${module.label.id}-"
  vpc_zone_identifier                   = data.aws_subnet_ids.all.ids
  max_size                              = "1"
  min_size                              = "1"
  desired_capacity                      = "1"

  launch_template = {
    id                                  = "aws_launch_template.default.id
    version                             = "$$Latest"
  }

  # terraform-null-label example used here: Set tags on ASG and EC2 Servers
  tags                                  = module.label.tags_as_list_of_maps
}
```

### Advanced Example 3

See [complete example](./examples/complete)

This example shows how you can pass the `context` output of one label module to the next label_module,
allowing you to create one label that has the base set of values, and then creating every extra label
as a derivative of that.

```hcl
module "label1" {
  source      = "git::https://github.com/cloudposse/terraform-null-label.git?ref=master"
  namespace   = "CloudPosse"
  environment = "UAT"
  stage       = "build"
  name        = "Winston Churchroom"
  attributes  = ["fire", "water", "earth", "air"]
  delimiter   = "-"

  label_order = ["name", "environment", "stage", "attributes"]

  tags = {
    "City"        = "Dublin"
    "Environment" = "Private"
  }
}

module "label2" {
  source    = "git::https://github.com/cloudposse/terraform-null-label.git?ref=master"
  context   = module.label1.context
  name      = "Charlie"
  stage     = "test"
  delimiter = "+"

  tags = {
    "City"        = "London"
    "Environment" = "Public"
  }
}

module "label3" {
  source    = "git::https://github.com/cloudposse/terraform-null-label.git?ref=master"
  name      = "Starfish"
  stage     = "release"
  context   = module.label1.context
  delimiter = "."

  tags = {
    "Eat"    = "Carrot"
    "Animal" = "Rabbit"
  }
}
```

This creates label outputs like this:

```hcl
label1 = {
  "attributes" = [
    "fire",
    "water",
    "earth",
    "air",
  ]
  "delimiter" = "-"
  "id" = "winstonchurchroom-uat-build-fire-water-earth-air"
  "name" = "winstonchurchroom"
  "namespace" = "cloudposse"
  "stage" = "build"
}
label1_context = {
  "additional_tag_map" = {}
  "attributes" = [
    "fire",
    "water",
    "earth",
    "air",
  ]
  "delimiter" = "-"
  "enabled" = true
  "environment" = "uat"
  "label_order" = [
    "name",
    "environment",
    "stage",
    "attributes",
  ]
  "name" = "winstonchurchroom"
  "namespace" = "cloudposse"
  "regex_replace_chars" = "/[^a-zA-Z0-9-]/"
  "stage" = "build"
  "tags" = {
    "Attributes" = "fire-water-earth-air"
    "City" = "Dublin"
    "Environment" = "Private"
    "Name" = "winstonchurchroom"
    "Namespace" = "cloudposse"
    "Stage" = "build"
  }
}
label1_tags = {
  "Attributes" = "fire-water-earth-air"
  "City" = "Dublin"
  "Environment" = "Private"
  "Name" = "winstonchurchroom"
  "Namespace" = "cloudposse"
  "Stage" = "build"
}
label2 = {
  "attributes" = [
    "fire",
    "water",
    "earth",
    "air",
  ]
  "delimiter" = "+"
  "id" = "charlie+uat+test+firewaterearthair"
  "name" = "charlie"
  "namespace" = "cloudposse"
  "stage" = "test"
}
label2_context = {
  "additional_tag_map" = {}
  "attributes" = [
    "fire",
    "water",
    "earth",
    "air",
  ]
  "delimiter" = "+"
  "enabled" = true
  "environment" = "uat"
  "label_order" = [
    "name",
    "environment",
    "stage",
    "attributes",
  ]
  "name" = "charlie"
  "namespace" = "cloudposse"
  "regex_replace_chars" = "/[^a-zA-Z0-9-]/"
  "stage" = "test"
  "tags" = {
    "Attributes" = "firewaterearthair"
    "City" = "London"
    "Environment" = "Public"
    "Name" = "charlie"
    "Namespace" = "cloudposse"
    "Stage" = "test"
  }
}
label2_tags = {
  "Attributes" = "firewaterearthair"
  "City" = "London"
  "Environment" = "Public"
  "Name" = "charlie"
  "Namespace" = "cloudposse"
  "Stage" = "test"
}
label3 = {
  "attributes" = [
    "fire",
    "water",
    "earth",
    "air",
  ]
  "delimiter" = "."
  "id" = "starfish.uat.release.firewaterearthair"
  "name" = "starfish"
  "namespace" = "cloudposse"
  "stage" = "release"
}
label3_context = {
  "additional_tag_map" = {}
  "attributes" = [
    "fire",
    "water",
    "earth",
    "air",
  ]
  "delimiter" = "."
  "enabled" = true
  "environment" = "uat"
  "label_order" = [
    "name",
    "environment",
    "stage",
    "attributes",
  ]
  "name" = "starfish"
  "namespace" = "cloudposse"
  "regex_replace_chars" = "/[^a-zA-Z0-9-]/"
  "stage" = "release"
  "tags" = {
    "Animal" = "Rabbit"
    "Attributes" = "firewaterearthair"
    "City" = "Dublin"
    "Eat" = "Carrot"
    "Environment" = "uat"
    "Name" = "starfish"
    "Namespace" = "cloudposse"
    "Stage" = "release"
  }
}
label3_tags = {
  "Animal" = "Rabbit"
  "Attributes" = "firewaterearthair"
  "City" = "Dublin"
  "Eat" = "Carrot"
  "Environment" = "uat"
  "Name" = "starfish"
  "Namespace" = "cloudposse"
  "Stage" = "release"
}

```






## Makefile Targets
```
Available targets:

  help                                Help screen
  help/all                            Display help for all targets
  help/short                          This help short screen
  lint                                Lint terraform code

```
## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|:----:|:-----:|:-----:|
| additional_tag_map | Additional tags for appending to each tag map | map(string) | `<map>` | no |
| attributes | Additional attributes (e.g. `1`) | list(string) | `<list>` | no |
| context | Default context to use for passing state between label invocations | object | `<map>` | no |
| delimiter | Delimiter to be used between `namespace`, `environment`, `stage`, `name` and `attributes` | string | `-` | no |
| enabled | Set to false to prevent the module from creating any resources | bool | `true` | no |
| environment | Environment, e.g. 'prod', 'staging', 'dev', 'pre-prod', 'UAT' | string | `` | no |
| label_order | The naming order of the id output and Name tag | list(string) | `<list>` | no |
| name | Solution name, e.g. 'app' or 'jenkins' | string | `` | no |
| namespace | Namespace, which could be your organization name or abbreviation, e.g. 'eg' or 'cp' | string | `` | no |
| regex_replace_chars | Regex to replace chars with empty string in `namespace`, `environment`, `stage` and `name`. By default only hyphens, letters and digits are allowed, all other chars are removed | string | `/[^a-zA-Z0-9-]/` | no |
| stage | Stage, e.g. 'prod', 'staging', 'dev', OR 'source', 'build', 'test', 'deploy', 'release' | string | `` | no |
| tags | Additional tags (e.g. `map('BusinessUnit','XYZ')` | map(string) | `<map>` | no |

## Outputs

| Name | Description |
|------|-------------|
| attributes | List of attributes |
| context | Context of this module to pass to other label modules |
| delimiter | Delimiter between `namespace`, `environment`, `stage`, `name` and `attributes` |
| environment | Normalized environment |
| id | Disambiguated ID |
| label_order | The naming order of the id output and Name tag |
| name | Normalized name |
| namespace | Normalized namespace |
| stage | Normalized stage |
| tags | Normalized Tag map |
| tags_as_list_of_maps | Additional tags as a list of maps, which can be used in several AWS resources |


## Copyright

Copyright © 2017-2019 [Cloud Posse, LLC](https://cpco.io/copyright)



## License 

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0) 

See [LICENSE](LICENSE) for full details.

    Licensed to the Apache Software Foundation (ASF) under one
    or more contributor license agreements.  See the NOTICE file
    distributed with this work for additional information
    regarding copyright ownership.  The ASF licenses this file
    to you under the Apache License, Version 2.0 (the
    "License"); you may not use this file except in compliance
    with the License.  You may obtain a copy of the License at

      https://www.apache.org/licenses/LICENSE-2.0

    Unless required by applicable law or agreed to in writing,
    software distributed under the License is distributed on an
    "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
    KIND, either express or implied.  See the License for the
    specific language governing permissions and limitations
    under the License.
