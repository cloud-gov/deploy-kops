Baseline notes for the POC

* Baseline

Each time you need to interact with KOPS (not on jumphosts yet) you will need:

- working aws cli creds for your env
- a valid kube config file with a valid admin login token (kubectl works)
- KOPS ENV settings defined in your session

* KOPS ENV values

```
export AWS_ACCESS_KEY_ID=<AWS KEY>
export AWS_SECRET_ACCESS_KEY=<AWS SECRET>
export AWS_PROFILE=<AWS CLI CONFIG PROFILE NAME IF YOU WANT THAT USED>
export KOPS_STATE_STORE=s3://<ENV S3 bucket>
export NAME=<name of KOPS cluster>
export AWS_REGION=<aws govcloud region>
export KOPS_AWS_REGION=<aws govcloud region>
```

* KOPS Validate Command

```
kops validate cluster
```


* Rolling Updates

KOPS has a rolling update feature when you need to change out a config item, like AMI, etc

Change out node features as example:

```
kopes edit instancegroup nodes
```

This opens a vi editor after it DLs the file from the S3 bucket.  You can look under the `validate` command from above the other instance groups

You can then roll you changes via:

```
kops rolling-update ${NAME} --yes
```

* SSH into nodes

As part of the deployment of KOPS you create a ssh key pair - you can use that priv key from that pair to ssh into any of the masters or nodes using user `admin`
