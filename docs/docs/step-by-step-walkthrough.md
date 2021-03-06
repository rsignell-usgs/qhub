# Step by Step QHub Cloud Deployment

This guide makes the following assumptions:

- [Github actions] will be used for CICD
- Oauth will be via [auth0]
- DNS registry will be through [Cloudflare]

Other providers can be used, but you will need to consult their documention on setting up oauth and DNS registry. If a different provider is desired, then the corresponding flag in the `qhub` command line argument can be ommited.

## 1. Installing QHub:

- Via github:

        git clone git@github.com:Quansight/qhub.git
        cd qhub
        python ./setup.py install

- Via pip:

        pip install qhub

## 2. Environment variables

Several different environment variables must be set in order for deployment to be fully automated. The following subsections will describe the purpose and method for obtaining the values for these variables.

### 2.1 Cloudflare

Cloudflare will automate the DNS registration. First a [Cloudflare][Cloudflare_signup] account needs to be created and [domain name] registered through it. If an alternate DNS provider is desired, then omit the `--dns-provider cloudflare` flag for `qhub deploy`. At the end of deployement an IP address (or CNAME for AWS) will be output that can be registered to your desired URL.

Within your Cloudflare account, follow these steps to generate a token

- Click user icon on top right and click `My Profile`
- Click the `API Tokens` menu and select `Create Token`
- Click `Use Template` next to `Edit zone DNS`
- Under `Permissions` in addition to `Zone | DNS | Edit` use the `Add more` button to add `Zone | Zone | Read`, `Zone | Zone Settings | Read`, and `Account | Account Settings | Read` to the Permissions
- Under `Account Resources` set the first box to `Include` and the second to your desired account
- Under `Zone Resources` set it to `Include | Specific zone` and your domain name
- Click continue to summary
- Click Create Token
- `CLOUDFLARE_TOKEN`: Set this variable equal to the token generated

### 2.2 Auth0

After creating an [Auth0](https://auth0.com/) account and logging in, follow these steps to create a access token: 

- Click Applications on the left
- Click `Create Application`
- Select `Machine to Machine Applications`
- Select `Auth0 Management API` from the dropdown
- Click `All` next to `Select all` and click `Authorize`
- `AUTH0_CLIENT_ID`: Set this variable equal to the `Cliend ID` string under `Settings`
- `AUTH0_CLIENT_SECRET`: Set this variable equal to the `Client Secret` string under `Settings`
- `AUTH0_DOMAIN`: Set this variable to be equal to your account name (indicated on the upper right) appended with `.auth0.com`. IE an account called `qhub-test` would have this variable set to `qhub-test.auth0.com`

### 2.3 Github
Your github username and access token will automate the creation of the repository that will hold the infrastructure code as well as the github secrets

- `GITHUB_USERNAME`: Set this to your github username
- `GITHUB_TOKEN`: Set this equal to your [github access token]


### 2.4 Cloud Provider Credentials
The access key for the cloud providers require fairly wide permissions in order for QHub to deploy. As such, all testing for QHUb has been done with owner/admin level permissions. We welcome testing to determine the minimum required permissions for deployment [(open issue)](https://github.com/Quansight/qhub/issues/173).

#### 2.4.1 Amazon Web Services

Please see these instructions for [creating an IAM role] with admin permissions and set the below variables.

- `AWS_ACCESS_KEY_ID`: The public key for an IAM account
- `AWS_SECRET_ACCESS_KEY`: The Private key for an IAM account
- `AWS_DEFAULT_REGION`: The region where you intend to deploy QHub

#### 2.4.2 Digital Ocean

In order to get the DigitalOcean access keys follow this [digitalocean tutorial].

- `DIGITALOCEAN_TOKEN`: Follow [these instructions] to create a digital ocean token 
- `SPACES_ACCESS_KEY_ID`: Follow [this guide] to create a spaces access key/secret
- `SPACES_SECRET_ACCESS_KEY`: The secret from the above instructions
- `AWS_ACCESS_KEY_ID`: Due to a [terraform] quirk, set to the same as `SPACES_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`: Due to the quirk, set to the same as `SPACES_SECRET_ACCESS_KEY`

#### 2.4.3 Google Cloud Platform

Follow [these detailed instructions] on creating a Google service account with owner level permissions. With the service account created, follow [these steps] to create and download a json credentials file.

- `GOOGLE_CREDENTIALS`: Set this to the path to your credentials file
- `PROJECT_ID`: Set this to the project ID listed on the home page of your Google console under `Project info`

We're done with the hardest part of deployment!

### 2.5 QHub init

The next step is to run `qhub init` to generate the configuration file `qhub-config.yaml`. This file is where the vast majority of tweaks to the system will be made. There are are several optional (yet highly recommended) flags that deal with automating the deployment:

- `--project`: Chose a name that [is a compliant name for S3/Storage buckets]. IE `test-cluster`
- `--domain`: This is the base domain for your cluster. After deployment, the DNS will use the base name prepended with `jupyter`. IE if the base name is `test.qhub.dev` then the DNS will be provisioned as `jupyter.test.qhub.dev`. This pattern is also applicable if you are setting your own DNS through a different provider.
- `--ci-provider`: This specifies what provider to use for ci-cd. Currently github-actions is supported.
- `--oauth-provider`: This will set configuration file to use auth0 for authentication
- `--oauth-auto-provision`: This will automatically create and configure an auth0 application
- `--repository`: The repository name that will be used to store the infrastructure as code
- `--repository-auto-provision`: This sets the secrets for the github repository

Best practices is to create a new directory and run all the qhub commands inside of it. An example of the full command is below:

`qhub init gcp --project test-project --domain test.qhub.dev --ci-provider github-actions --oauth-provider auth0 --oauth-auto-provision --repository github.com/quansight/qhub-test --repository-auto-provision`

### 2.6 QHub render

After initializing, we then need to create all of terraform configuration. This done by running:

        qhub render -c qhub-config.yaml ./ -f
        
You will notice that several directories are created: 
  - `environments` : conda environments are stored
  - `infrastructure` : terraform files that declare state of infrastructure
  - `terraform-state` : required by terraform to securly store the state of the terraform deployment
  - `image` : docker images used in qhub deployment including: jupyterhub, jupyterlab, and dask-gateway
        
### 2.7 QHub deploy

Finally we can deploy with:

    qhub deploy -c qhub-config.yaml --dns-provider cloudflare --dns-auto-provision

The terminal will prompt to press `[enter]` to check oauth credentials (which were added by qhub init). After pressing `enter` the deployment will continue and take roughly 10 minutes. Part of the output will show an "ip" address (DO/GCP) or a CNAME "hostname" (AWS) based on the the cloud service provider:

    Digital Ocean/Google Cloud Platform:
       
        Outputs:

        ingress_jupyter = {
        "hostname" = ""
        "ip" = "xxx.xxx.xxx.xxx"
        }

    AWS:       
        Outputs:

        ingress_jupyter = {
        "hostname" = "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx-xxxxxxxxxx.us-east-1.elb.amazonaws.com"
        "ip" = ""
        }

        
### 2.8 Push repository

Add all files to github:

    git add .github/ .gitignore README.md environments/ image/ infrastructure/ qhub-config.yaml  terraform-state/

Push the changes to your repo:

    git push origin master


### 3 Post github deployment:

After the files are in github all CICD changes will done via github actions and triggered by a commit to master. To use gitops, make a change to the `qhub-ops.yaml` in a new branch and create pull request into master. When the pull request is merged, it will trigger a deployement of all of those changes to your QHub.

The first thing you will want to do is add users to your new QHub. Any type of supported authorization from auth0 can be used as a username. Below is an example configuration of 2 users:

        joeuser@example:
            uid: 1000000
            primary_group: users
            secondary_groups:
                - billing
                - admin
        janeuser@example.com:
            uid: 1000001
            primary_group: users

As seen above, each username has a unique `uid` and a `primary_group`. Optional `secondary_groups` may also be set for each user.

### 4 Git ops enabled

Since the infrastructure state is reflected in the repository, it allows self-documenting of infrastructure and team members to submit pull requests that can be reviewed before modifying the infrastructure.

Congratulations! You have now completed your QHub cloud deployment!

[Github actions]: https://github.com/features/actions
[via github]: https://docs.github.com/en/free-pro-team@latest/developers/apps/authorizing-oauth-apps
[auth0]: https://auth0.com/
[Cloudflare]: https://www.cloudflare.com/
[AWS Environment Variables]: https://github.com/Quansight/qhub/blob/ft-docs/docs/docs/aws/installation.md
[Digital Ocean Environment Variables]: https://github.com/Quansight/qhub/blob/ft-docs/docs/docs/do/installation.md
[Google Cloud Platform]: https://github.com/Quansight/qhub/blob/ft-docs/docs/docs/gcp/installation.md
[Cloudflare_signup]: https://dash.cloudflare.com/sign-up
[domain name]: https://www.cloudflare.com/products/registrar/
[github_oath]: https://developer.github.com/apps/building-oauth-apps/creating-an-oauth-app/
[doctl]: https://www.digitalocean.com/docs/apis-clis/doctl/how-to/install/
[oauth application]: https://docs.github.com/en/free-pro-team@latest/developers/apps/authorizing-oauth-apps
[recording your DNS]: https://support.cloudflare.com/hc/en-us/articles/360019093151-Managing-DNS-records-in-Cloudflare
[github access token]: https://docs.github.com/en/free-pro-team@latest/github/authenticating-to-github/creating-a-personal-access-token
[creating an IAM role]: https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create.html
[digitalocean tutorial]: https://www.digitalocean.com/community/tutorials/how-to-create-a-digitalocean-space-and-api-key
[these instructions]: https://www.digitalocean.com/docs/apis-clis/api/create-personal-access-token/
[this guide]: https://www.digitalocean.com/community/tutorials/how-to-create-a-digitalocean-space-and-api-key
[terraform]: https://www.terraform.io/
[these detailed instructions]: https://cloud.google.com/iam/docs/creating-managing-service-accounts
[these steps]: https://cloud.google.com/iam/docs/creating-managing-service-account-keys#iam-service-account-keys-create-console
[is a compliant name for S3/Storage buckets]: https://www.google.com/search?q=s3+compliant+name&oq=s3+compliant+name&aqs=chrome..69i57j0.3611j0j7&sourceid=chrome&ie=UTF-8
