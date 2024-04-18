# Terminology

How terraform works?

Written in Go lang and interfaces with the api (crud ops) of the provider (aws, azure, etc)

Workflow
1. Write code
2. Run Terraform plan to validate syntax, etc
3. Run apply
    Commit to Git
    Cicd tool
    Deploy    


Terraform State => Stores information about what's deployed (what's running)

Declarative Vs Procedural

Declarative - What you want the final deployment to look like? Requires state
Procedural - How you want to deploy resources? Doesn't require state

# Starting with Terraform
* Create the aws account and the cloud9 ec2 instance
* Go to terraform and check the yum command to install it in the cloud9
* Create a terraform-docker folder and inside a main.tf file inside.
* In the file we need to write the docker provider and instantiate it, like so:

    terraform {
        required_providers{
            docker = {
                source = "kreuzwerker/docker"
            }
        }
    }

    provider "docker" {}

* run the command "terraform init"
* remember that a hidden folder .terraform will be created and its huge so be sure to add it to the git ignore file or deleted before commit.


# Versioning
* In the .terraform folder there's a lock file with all the dependencies' versions. 
* To downgrade we must add the version to the docker provider and then not init command but "terraform init -upgrade" 
* In typical software versioning ex "2.12.9" major.minor.patch, if we add the symbol ~> befor the version we limit the minor version.ex: 

    terraform
    ....
    docker = ...
    version = "~> 2.12.0"

This will limit the version to 2.12.x , if we remove the last digit like 2.12, it will limit the major version "2".

# Getting and adding resources

* on the main.tf file add the following:

    resource "docker_image" "nodered_image" {
    name = "nodered/node-red:latest"
    }

* To see the changes from what is initialized and what is written in the file we can input the command "terraform fmt -diff".

* Now we're going to validate the file by inputing the command "terraform plan".

* After validating all is ok and it is in fact what we want to do, input the command "terraform apply" and then we're prompt again to check if everything is ok, write "yes".

* Now terraform begins by downloading the image and then creating the resource.

* To check the image is downloaded input "docker image ls" and check the list for the resource

* Now every "terraform plan" that we do, will revert the current state back to the initial plan, creating or deleting resources to match the plan.

* To destroy the current state we input "terraform destroy".

* To save the plan we specify the -out arg as: terraform plan -out=plan1 <br>
This will create an encoded file.

* To apply a certain plan we: terraform apply plan1 <br> 
This will not show a confirmation dialog (great for automation)

# Deploy docker container and referencing other resources

* To add a container as a resource:

        resource "docker_container" "nodered_container"{
        name = "nodered" //this is an invented name to be able to ref later
        image = docker_image.nodered_image.image_id
        ports {
            internal = 1880
            external = 1880
        }
    }

* After this, input again the terraform plan command, check if ev is ok
* Input terraform apply and after checking if ev ok write "yes"
* Now a terraform.tfstate was created and we can check if the container is online by docker ps and checking the list.
* To show all the terraform resources input the command "terraform state list"


# Accesing the container

* first we need to whitelist our pc ip on the ec2 instance
* then in the browser input the ip of the instance followed by the external port nr.


# Console and Outputs
Allows to export and modify state info from our deployment to be displayed, consumed by other apps or by terraform modules.

* after apply
* to see the state (instead of opening the file) "terraform show"
* to search for something we can "terraform show | grep ip"
* to check something in the resources we can "terraform console" and in the CLI of terraform call the resources we want to check, ex: docker_container.nodered_container.network_data[0].ip_address
* we can output a value like so:

    output "IP-Address"{
        value = docker_container.nodered_container.network_data[0].ip_address
        description = "The ip address of the container"
    }

* and then to show the output we "terraform output"      

# Console functions
terraform console and then

## join

join => (separator, list) => (",", ["foo","bar"])

Ex: Output the ip adress of the container and the external port

join(":",[docker_container.nodered_container.network_data[0].ip_address,docker_container.nodered_container.ports[0].external])

"172.17.0.2:1880"

## random
Important for avoiding clashing of names, image names etc

* create a resource first like so:

        resource "random_string" "random" {
        length = 4
        special = false
        upper = false
    }

* as we added another resource we must "terraform init"
* terraform apply
* terraform state list to check the newly created resource
* together with the function join we add the random to the name of the resource like so:

    resource "docker_container" "nodered_container"{
    name = join("-",["nodered",random_string.random.result])
    ...

## count

Inside a resource, if you add the attribute to the random resource creation ex: count = 2, you'll add 2 instance of whatever you're creating.

        resource "random_string" "random" {
        count = 2
        ...
    }

and to create 2 similar containers that each gets a random of the 2 randoms that we created we use count index :

        resource "docker_container" "nodered_container"{
            count = 2
            name = join("-",["nodered",random_string.random[count.index].result])
        ...
        }

and to output them we can access its indexes like so:

    output "container-name1"{
        value = docker_container.nodered_container[0].name
        ...
    }


## splat
Works like a for each loop. Ex:

    docker_container.nodered_container[*].name

will output every nodered_container name available.

## for expressions
It's a for loop to loop through lists and it's syntax is:

    [for i in [1,2,3]: i+1]


## Tainting
Way to force a resource to be destroyed and reapplied. Most common reason for this is to reapply some sort of configuration. Its like rebooting.

    terraform taint random_string.random[0]

Now if we plan now, this resource will be marked to be destroyed and created and everything that depends on it.
To cancel it, we can send the opposite command:

    terraform untaint random_string.random[0]


# Import
A way of import a resource to the state, to be able to apply or destroy an ongoing resource that terraform doesn't know.

So for ex we have a container running out of state, to import it we can create a resource, add its name by searching docker ps -a and get the name.
add the image property also.
For the import we input the command:

    terraform import docker_container.nodered_container2 $(docker inspect --format="{{.ID}}" nodered-3pfl)

to check import run terraform state list.

## State remove
Let's say a container resource get's downed from other source than terraform. On the state there are 2 online but on reality only 1, meaning, our state becomes out of sync with our actual environment. If we don't want to apply, we just want to sync the state we can:

    terraform state rm resouce_name

this removes the resource from the state,


# Variables
You can write it on the main.tf as a resource or have a file with all the vars.
We can include also the validation block, that will check the bounds of that var when we plan or apply

        variable "int_port" {
            type = number
            default = 1880
        
            validation {
                condition = var.int_port == 1880
                error_message = "The internal port must be 1880."
            }
            }


to reference them:

    ...
        internal = var.int_port
        external = var.ext_port
    }


To have the vars and output in different files we:

* create new files variables.tf and outputs.tf
* cut and paste into those files
* terraform plan
* terraform --auto-approve && terraform apply --auto-approve

## Sensitive vars and tfvars files
If we have secure pass or sensitive information we:

* create a terraform.tfvars file
* paste the var there in the format name_of_the_var = value
* add the file to git ignore file. (VERY IMPORTANT! THIS IS CRUCIAL TO NOT UPLOAD CREDENTIALS OR SENSITIVE INFO TO GITHUB)

In case of multiple tfvars file we can selrct which one we'll be used by:

    terraform plan --var-file=nameosthefile.tfvars
    # or to address individual vars (overrides the tfvars file)
    terraform plan -var ext_port=1880

Despuite putting sensitive vars in a file, they can be output by a runner (cicd runner, github actions runner, etc), to deny that possibility we can add the argument sensitive=true when we're defining the var:


    variable "ext_port" {
    type = number
    ...
    sensitive = true
    ...}

Now we can't output them in the cli, because it gives an error. To overcome this in our output calls we can also add the argument sensitive = true and this will annotate the output value as sensitive as well:

    output "ip-address"{
        ...
        sensitive = true
    }

The output will look like this:

    ip-address = <sensitive>

# Local-exec for Bind-mount persistence

The local-exec provisioner invokes a local executable after a resource is created. This invokes a process on the machine running terraform, not on the resource.

We could add it to a random resource, but its more organized if we create a resource just for it. As we don't have a resource to create here we can create a null-resource that triggers the local-exec:

    resource "null_resource" "dockervol" {
    provisioner "local-exec" {
        command = "mkdir noderedvol/ || true && sudo chown -R 1000:1000 noderedvol/"
    }
    }

Note! The || true is to not throw an error in case the state gets out of sync (since the noderedvol folder is created by a command is not idempotent and not guaranteed by the state.) So in case we destroy (folder does not disappear), and then apply (state will run the command to create the folder), the command will error because the folder already exists. With the || true never throws.

Now we need to add the volume to the container, like so:

    resource "docker_container" "nodered_container"{
        count = var.container_count
        name = join("-",["nodered",random_string.random[count.index].result])
        image = docker_image.nodered_image.image_id
        ports {
            internal = var.int_port
            external = var.ext_port
        }
        volumes {
        container_path = "/data"
        host_path = "/home/ec2-user/environment/terraform-docker/noderedvol"
        }
    }

!NOTE - container_path = "/data" is the path described in the documentation of nodered; the host_path is the path to the volume folder. We can get the path by inputing on the console "pwd"

# Local Value
Like local vars but allow expressions and functions.

to set:

    locals {
        container_count = length(var.ext_port)
    }

to reference:

    resource "docker_container" "nodered_container"{
        count = local.container_count
        ...

## Min and Max function

We can do min and max functions with help of the expansion "..." which means treat it for each element in this list:

    validation {
        condition = max(var.ext_port...) <= 65535 && min(var.ext_port...) > 0
        error_message = "The external port must be the valid port range 0 - 65535."
    }  

## Path refs and String interpolation
It's not a best practice to hard code paths, so in our host_path we can make use of the root (or cwd) path ref and string interpolation to concat the path: 

        volumes {
            container_path = "/data"
            host_path = "${path.cwd}/noderedvol"
        }

# Maps and Lookups:the image variable to handle different environments

To be able to fine tune deployments for dev and prod we're going to use different node-red images. The dev will be the latest image with all features allowing easy development, but has a larger file size and larger attack surface. The prod image will be minimal with a smaller attack surface.

1. Create a var on variables.tf file called env and other called image.


    variable "env" {
    type = string
    description = "Env to deploy to" 
    default = "dev"
    }

    variable "image" {
    type = map
    description = "image for container"
    default = {
        dev = "nodered/node-red:latest"
        prod = "nodered/node-red:latest-minimal"
    }
    }

2. Now we need to set the image name depending on the env we're on with the help of the lookup function that returns a string:

    resource "docker_image" "nodered_image" {
    name = lookup(var.image, var.env)
    }

3. We need to apply the same logic to ports:

* In the tfvars file change the ext_ports to a map:

    ext_port = {
    dev = [1980, 1981]
    prod = [1880, 1881]
    }

* In the variables file change the type and add another validation creating a validation for each env ports:

    variable "ext_port" {
    type = map
    
    validation {
        condition = max(var.ext_port["dev"]...) <= 65535 && min(var.ext_port["dev"]...) >= 1980
        error_message = "The external port must be the valid port range 0 - 65535."
    }
    
    validation {
        condition = max(var.ext_port["prod"]...) < 1980 && min(var.ext_port["prod"]...) >= 1880
        error_message = "The external port must be the valid port range 0 - 65535."
    }
    }

* still on the variable file, change the locals to account for the map

    locals {
    container_count = length(lookup(var.ext_port,var.env))
    }

* on main, the external ports must now include a lookup to include the new type (map) of var:

    resource "docker_container" "nodered_container"{
        ...
        ports {
            internal = var.int_port
            external = lookup(var.ext_port, var.env)[count.index]
        }
        ...
    }


# Workspaces
They're isolated versions of the terraform state that allows multiple deployments versions of the same environment. (potentially with different configuration, counts and var definitions). Typically each workspace will be tied to a branch in source control. We can see it as a smaller replacement for environments.


1. To create:

    terraform workspace new dev

This will create a new folder terraform.tfstate.d with subfolder for the workspaces you just created.

2. To see in what workspace you're on:

    terraform workspace show

3. To see a list of workspaces:

    terraform workspace list

4. To switch workspaces:

    terraform workspace select dev

## Referencing your workspaces
Having a var telling the env is not best practice. We can achieve the same thing by referencing terraform.workspace.

1. Remove the env var
2. Wherever we have var.env we replace with terraform.workspace

## Map refs instead of lookups
We can now remove the lookup and replace it by the direct call of its reference like so:

    external = var.ext_port[terraform.workspace]


# Modules
Like the root module, is a set of Terraform configuration files in a single directory.
It's generally a best practice to have logical separation between images/container logic in modules.
So the root module only has the calls to the specific modules.

1. Create a providers.tf file in the root module and cut and paste the providers from main.tf
2. Create a folder called image to house the image module. 
3. Cd into the new folder and create the following files with the command:

    touch main.tf outputs.tf variables.tf providers.tf

4. Copy the providers from the root to the image modules.
5. Cut the image resource and paste it onthe the main.tf of the image module
6. On the main.tf of the root, create a new module call:

    module "image" {
        source = "./image"
    }

7. The container still references an image resource, to get around that we must create an output. Go to the  
image outputs.tf and add an output:

    output "image_out" {
        value = docker_image.nodered_image.image_id
    }

8. On the container resource, on the image attribute, reference the module and its output:

    module.image.image_out

9. As we added a new module we must cd .. out of the image module and then terraform init.


## Passing args and vars
After the attribute "source" of the module we can add args to be passed as so:

    module "image" {
        source = "./image"
        image_in = var.image[terraform.workspace]
    }

In this case the arg name is image_in and then we're passing the same value we were passing before.
Now, to receive it on the image module, on the main.tf we must receive it like so:

    resource "docker_image" "nodered_image" {
        name = var.image_in
    }

We must also define the var on the image module. Go to the image module on the variables.tf and define the var but add only the description, as the value is already being passed as an argument:

    variable "image_in" {
        description = "name of image"
    }      

# Terraform Graph
When coupled with a utility called Graphviz allows us to visuallize the deployment dependencies.

1. Install graphviz by sudo yum (or apt) install graphviz
2. terraform graph | dot -Tpdf > graph-plan.pdf

As we explore the dependencies map we found that the container has no dependency on the volume created. This must be address because if be any reason the volume takes longer to be created, the container will fail because it doesn't have a volume to write into. So we can adress it implicity (we can create on the container a false necessity to include something that the process of building the volume will produce, like an id), or we can add a explicit dependency as such:

    resource "docker_container" "nodered_container"{
        depends_on = [null_resource.dockervol]
        ...


# Docker Volume

!The following is a bad way to do it. It just to explain if we create a volume this way, when we destroy the volumes will continue to exit.

1. First let's clean up the null resource, it's not best practice so delete it.
2. Remove the depends-on attribute on the container module as well.

* We could create a module for each volume, but in this case we want the container and the volume coupled (when the container is destroyed we want the volume destroyed)

3. On the main.tf of the container, on the volumes attribute:

    volumes {
      container_path = var.container_path_in
      volume_name = "${var.name_in}-volume"
    }

4. docker volume ls to list the volumes we just created

5. to check if the files are on the volumes "sudo su" and then ls /var/lib/docker/volumes/nodered-dev-hdjd-volume/_data (name of the container will be different) and then "exit" to exit sudo mode.

! The right way to achieve this is:

1. Create a volume resource on the main.tf of the container and reference it in the container "volume_name" attribute:

    resource "docker_container" "nodered_container"{
        ...
        volumes {
        ...
        volume_name = docker_volume.container_volume.name
        }
    }

    resource "docker_volume" "container_volume" {
        name = "${var.name_in}-volume"
    }

2. Now that the volumes are withing terraform control, we can choose to block volume resource from being destroyed. In the docker volume resource add the following block:

    resource "docker_volume" "container_volume" {
        name = "${var.name_in}-volume"
        lifecycle {
            prevent_destroy = false
        }
    }


And now when we try destroy all the resources, if throws an error.
But we can destroy only targeted resources, as such:

    terraform destroy -target=nameOfTheResource


# Self Referencing and Provisioner Failure Modes
How to backup our container volumes when we destroy our infrastructure.

1. On the volume resource, create an attribute where we'll define our backup folder

        resource "docker_volume" "container_volume" {
        count = var.count_in
        name = "${var.name_in}-${random_string.random[count.index].result}-volume"
        lifecycle {
            prevent_destroy = false
        }
        provisioner "local-exec" {
            when = destroy
            command = "mkdir ${path.cwd}/../backup/"
            on_failure = continue
        }
    }

2. Create another provision to actually manage the backups

    ...

    provisioner "local-exec" {
        when = destroy
        command = "sudo tar -czvf ${path.cwd}/../backup/${self.name}.tar.gz ${self.mountpoint}/"
        on_failure = fail
    }