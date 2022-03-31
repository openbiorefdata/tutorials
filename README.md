Tutorials provided below showcase how to make use of the [Open Bio Reference
Data](https://github.com/openbiorefdata/documentation) in
[Galaxy](https://galaxyproject.org/) and
R/[Bioconductor](https://bioconductor.org/) applications.

# Reference data in Galaxy

## Configuring Galaxy

To make use of the reference data available in this repository, it is necessary
to install Galaxy. There are several ways to [install
Galaxy](https://getgalaxy.org/) and this repository can be used with any of them
(assuming your platform has an available CVMFS client). The method described
here is using the [Galaxy Helm
chart](https://github.com/galaxyproject/galaxy-helm). We will create a cluster
on AWS, deploy Galaxy, and configure it to use this data repository for
reference data.

### Create an Amazon EKS cluster

[EKS](https://aws.amazon.com/eks/) provides a managed Kubernetes cluster that
can be dynamically scaled for your Galaxy workloads. To create a cluster, make
sure you have appropriate [IAM
role](https://docs.aws.amazon.com/eks/latest/userguide/service_IAM_role.html)s
and the [AWS
CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
configured, as well as have [kubectl](https://kubernetes.io/docs/tasks/tools/),
[helm](https://helm.sh/docs/intro/install/),
[eksctl](https://eksctl.io/introduction/#installation), and
[Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html)
commands installed. Next, download this [cluster configuration
file](https://github.com/galaxyproject/gxabm/blob/8267865642c4f3fd916e752835c358a7376896a7/build_aws_cluster.yaml),
and feel free to edit it to reflect your desired cluster size or machine type.
Then create a cluster via the following command:

`eksctl create cluster -f build_aws_cluster.yaml`

Note that this command will take about 20 minutes to complete. Once done, you
can interact with your cluster using the _kubectl_ and _helm_ commands.

### Deploy Galaxy

To deploy all the cluster resources required for Galaxy, we provide a small
Ansible playbook. Download the [playbook
file](https://github.com/galaxyproject/gxabm/blob/dev/setup_galaxy_on_cluster.yaml)
and the associated Galaxy [configuration values
file](https://github.com/galaxyproject/gxabm/blob/dev/galaxy_values.yaml). Look
over the files to make sure the disk sizes and other configurations match your
needs. Then run the playbook to install Galaxy:

`ansible-playbook setup_galaxy_on_cluster.yaml`

The above command will deploy all the components required for Galaxy, including
configuration of the reference data in this repository. To inspect the data, you
can `kubectl exec` into a Galaxy web container and navigate to the
`/cvmfs/data.galaxyproject.org/` directory. To access Galaxy, visit the URL
shown at the end of the output from above command.

### Make use of reference data

The configuration files in this repository, coupled with the Galaxy Helm chart,
make the reference data available to Galaxy tools that require them. Examples
where this configuration can be tested are tools such as BWA or Bowtie2. For
those tools, the available genomes are visible on the tool form under _Using
reference genome_ heading. To make use of the references, simply run a tool.
There are numerous tutorials available on the [Galaxy Training
website](https://training.galaxyproject.org/) with step-by-step instructions.

# Data packages in Bioconductor / R

Packages from the AWS Open Data repository can be used in any R session as the
CRAN-like repository makes thousands of data and software packages available to
users.

Moreover, the repository will be particularly useful for cloud-based instances
of R, typically using the popular _RStudio_ web-based client. The cloud
deployment can notably mount the entirety of the repository using tools such as
[RClone-CSI](https://github.com/wunderio/csi-rclone), thus making pre-compiled
packages ready for immediate use with the [containerized Bioconductor
image](https://github.com/Bioconductor/bioconductor_docker), particularly when
launched in Kubernetes via the [Bioconductor helm
chart](https://github.com/Bioconductor/bioconductor-helm).

## Deploying cloud-based R/Bioconductor in AWS

We encourage the use of [EKS](https://aws.amazon.com/eks/), which provides a
managed Kubernetes cluster in AWS, and for which we provide an example [helm
configuration
file](https://github.com/Bioconductor/bioconductor-helm/blob/devel/examples/eks-vals.yaml).

### Creating an EKS cluster

To create a cluster, make sure you have an appropriate [IAM
role](https://docs.aws.amazon.com/eks/latest/userguide/service_IAM_role.html)
and [AWS
CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
configured, as well as [kubectl](https://kubernetes.io/docs/tasks/tools/),
[helm](https://helm.sh/docs/intro/install/), and
[eksctl](https://eksctl.io/introduction/#installation) commands installed.

Then, create a cluster via the following command:

`eksctl create cluster --name=my-cluster --nodes=1 --node-volume-size=50`

Note that this command will take a few minutes to complete. Once done, you can
interact with your cluster using the _kubectl_ and _helm_ commands.

### Deploying _Bioconductor_ Helm Chart

Next, one can install the Bioconductor helm chart, with the provided example
configuration file:

```
helm install mybioc bioconductor
    --repo https://github.com/Bioconductor/helm-charts/raw/devel
    -f https://raw.githubusercontent.com/Bioconductor/bioconductor-helm/devel/examples/eks-vals.yaml
```

If this is your first time running the chart, keep in mind that it will take a
few minutes for the container images to be pulled and extracted.

If you wish to see events, which particularly will contain information on image
pulling and disk attachments, run:

`kubectl get events`

Finally, you can wait until the deployment is ready by running:

`kubectl wait --for=condition=available --timeout=600s deployment/mybioc-bioconductor`

Once it is up, you can get the IP/URL at which the cloud-based instance is
available by running:

`kubectl get svc mybioc-bioconductor`

## Using data packages in cloud-based _R / Bioconductor_

Use of data packages in the deployed cloud-based environment is
straight-forward. The environment has been configured to know (via the
BioCmirror option) the location of the AWS Open Data-hosted packages. Users can
discover available packages with the simple command

```
	BiocManager::available()        ## All AWS Open Data-hosted packages
	BiocManager::available("TCGA")  ## Packages with 'TCGA' in their name
```

Cloud-based packages are installed and ready for use via standard _R_ commands
with

```
	BiocManager::install("<package-name>")
	library("<package-name>")
```

Additionally, data in the Bioconductor AnnotationHub and ExperimentHub can be
seamlessly used with those packages, as the Hub central databases will point by
default to the AWS Open Data bucket.

## Advanced use: deploying Bioconductor Helm Chart with RClone

For advanced users who heavily rely upon a large number of data and software
packages, as well as for workshops and other concentrated use, one can
additionally pre-load the repository and have a cloud-based instance in which
all packages are readily available with no need for installation.

For example, one can use _[RClone](https://github.com/wunderio/csi-rclone) _to
premount the repository as follows:

```
helm upgrade --install rclone-csi csi-rclone
    --repo "https://github.com/CloudVE/helm-charts"
    --namespace csi-drivers –create-namespace
    --set storageClass.name="rclone"
    --set params.remote="s3"
    --set params.remotePath="v1/bioconductor/3.14"
    --set params.s3-provider="aws"
    --set params.s3-endpoint="https://awsopendata.bioconductor.org"
```

One can then configure the Bioconductor helm chart to make use of this storage
class via the RClone example values file, as follows:

```
helm install mybioc bioconductor
    --repo https://github.com/Bioconductor/helm-charts/raw/devel
    -f https://raw.githubusercontent.com/Bioconductor/bioconductor-helm/devel/examples/eks-vals.yaml
    -f https://raw.githubusercontent.com/Bioconductor/bioconductor-helm/devel/examples/eks-vals.yaml
```

Once it is up, you can get the IP/URL at which the cloud-based instance is
available by running:

`kubectl get svc mybioc-bioconductor`

In this case, use of data packages in the deployed cloud-based environment is
immediate, skipping the installation step.

`library("<package-name>")`
