# What can cloudfairy do for you?

# Prequisites
- node.js version 16 and up
- docker

# Optional
The following dependencies can be installed later, depends on how you use cloudfairy
- brew
- aws cli
- gcloud cli
- azure cli
- k3d
- terraform / openTofu
- terragrunt
- task

# Installation
```bash
# Install from the npm repository
npm i -g @cloudfairy/cli
```

# Initialize cloudfairy into any project: A Quick guide
```bash
# In your root project's folder
fairy init
```
Name your infrastructure. Use lower-case letters, avoid numerics, dashes and underscores.

## What's going on?
Cloudfairy has setup a `.cloudfairy` folder with an initial config file, lifecycle task files, and downloaded the public core module library.

# Let's model an example infrastructure for a three-tier-application
## Step 1: Adding a front-end component
```bash
fairy add
```
Select "Public Website". Name it "frontend".

Answer simple configuration questions or skip and use the defaults

Note that cloudfairy detects missing dependencies at this point, but we can ignore this for now.
The expected result would be similar to `frontend tzoziq0jsc`.

## Step 2: Let's add an api service
```bash
fairy add
```
Select "Application", name it "api".

Set up ingress (exposed to internet) to true as the service receives http requests

Set up the port you are listening to and extra environment variables if needed


## Step 3: Let's add database
```bash
fairy add
```
Select "Docker from Dockerhub". Name it "database".

For image we can select `mongo`.

Let's set up the port to be `27017` as this is the default mongo port.

Additional variables may be required, if we want quick access:
`MONGO_INITDB_ROOT_USERNAME=admin,MONGO_INITDB_ROOT_PASSWORD=pass`
> Please note: It is not safe to store secrets and passwords. In this example for the sake of simplicity we are not using secrets or tokens.

# Let's review what we have now
We have a simple modeling of your application. A database, an api service and a database.

The api needs to reach for the database, and cloudfairy provide "connections".

Not every component is connectable. A connection can be made if clodufairy understands pairing of two components.

```bash
fairy connect api database
```

This will look up in the module library for **connectors** that accepts "Application" as incoming and "Docker from dockerhub" as output.

> Connectors can have multiple interfaces, and components can receive multiple types of connections

When cloudfairy finds the connector, it will ask you for the connector's configuration: In our case, the name of the environment variables that would be injected to our api, the hostname of the database, and the port. Let's name them "DB_HOST" and "DB_PORT"

# Let's review our model
```bash
fairy ls
```

We expect a result looking similar to this:
```text
Project Info: example
Components: 3
  ┌──────────┬───────────────────────────┬──────────────┐
  │ Name     │ Type                      │ Connected to │
  ├──────────┼───────────────────────────┼──────────────┤
  │ frontend │ cloudfairy/public-website │              │
  │ api      │ cloudfairy/application    │     database │
  │ database │ cloudfairy/docker         │              │
  └──────────┴───────────────────────────┴──────────────┘
```

## Let's get a visual representation of our model
```bash
fairy tools graph
```

We can expect an output similar to this:
```text
,--------.  ,---.
|frontend|  |api|
|--------|  |---|
`--------'  `---'
               |
               |
          ,--------.
          |database|
          |--------|
          `--------'
```

# Let's deploy!
We will now deploy this locally, on our machine.
cloudfairy will use **k3d** to provision a k3s cluster on our machine, a docker registry, and will add all manifests.

The frontend will be deployed with an nginx
The api will be deployed with an image name: "api:dev" using a local docker registry, and an ingress will be set up.

The database will be deployed also, using our local `data` folder for persistency.

This operation may take a few minutes.

```bash
fairy apply
```

cloudfairy will pick from the library all the required modules. Every module that requires a dependency will invoke an automatic generation of the missing dependencies. In our case the "cluster" module will be added, the "role" module and the "cloud provider" module.

Cloudfairy supports manual resolution of these dependencies, as we can add the cluster and configure it as we will instead of defaults.

# What is going on?
cloudfairy generated a `.cloudfairy.build` folder with terragrunt artifact. It copied all the required terraform files from the library into `tf-modules` and generated a "dev" environment infrastructure. Cloudfairy modules support multiple cloud providers, and the **target cloud** can be either local (using k3d), GCP, AWS, Azure.

# Why do I need this?
cloudfairy is a virtual cloud designer. The devOps team can provide a library of modules that targets any cloud, and their developer environment equivalents. A module can be a single terraform file with data and resources, a bundle of terraform files or references to external modules. Anything that terraform supports.
The dependency system recognizes the missing pieces and pulls from the library the dependencies that were not manually added to the model.

The team exposes a set of properties that are reflected back into variables. cloudfairy module outputs can be used by connectors for another modules.

# Let's see the benefits for our developer teams
```bash
fairy devtool
```

This will fire up a graphical user interface, with a library inspector and the visual model of the application. It's and editor. Move things around, connect, disconnect, and change the configuration as you like. This UI replaces the base interactions of modeling with the cli.

Try adding another dockerhub image, connect the api to it, change the properties as you like.

# Deployment to our target cloud
The example library can be deployed to the google cloud platform, assuming you have the right credentials.

Let's create a new environment based on our current model
```
fairy env stg
```

Let's set target cloud provider to GCP.
```bash
fairy project set-cloud-provider gcp
```
Let's answer "Yes"

Before deploying to the cloud, we need to configure things a little bit more.
```bash
fairy preflight
```
This will test missing configurations required to deploy to the cloud.
An error message is expected: `Error Missing PROJECT_ID in project global config`

Let's configure this now:
```bash
fairy conf --global PROJECT_ID <your-gcp-project-id>
```
The next error will expect us the configure the remote state bucket for terraform. You now need to create a bucket or use an existing one for the terraform state.

The `gcp-cloud-provider` component requires you to configure the region, and `fairy preflight` will catch that.

Once all is set and good to go, assuming **you do have credentials, and gcloud is logged in**, you can run `fairy apply`.

This process will now take longer.
In GCP target, the **cluster** module requires **networking** and **IAM** and all these will be generated into your cloud and managed by terraform.

There is no need to keep the `.cloudfairy.build` artifact in your source code, as it will **always** generate the same result and file naming. Changes to the project itself will reflect and keep back track of deleted modules, and changesets so terraform can delete unused resources.

# What next?
- Learn how to build your own modules for quick-reuse and modeling with cloudfairy.
- Learn how cloudfairy works with tasks
- Learn how to add plugins, to integrate with external systems, such as `backstage`, pricing calculators and other means.
- Learn how to write your own plugins




