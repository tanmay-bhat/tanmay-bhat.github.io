---
layout: post
title: Switch Between Multiple AWS Profiles In Terminal Using ZSH Functions
date: 2023-12-08
tags: ["AWS", "linux"]
---

ZSH functions are a great way to automate tasks and make your life easier. I’ve been using ZSH for a while now, and I recently started using functions to automate tasks that I do on a daily basis.

Every day, I routinely switch between various AWS profiles, depending on thr account and the role that I’m working on. So I wanted to make it easier to switch between profiles.

In this blog, we’ll see how we can use ZSH functions to switch between AWS profiles. Along with customizing the prompt to show the current AWS profile that we’re using so that we are sure on the account before deleting a database.

Pre-requisites:
- OhMyZsh (If you’re not using OhMyZsh, you should :P)
- AWS CLI

## Configure AWS CLI with different profiles

First, we need to make sure that we have configured the AWS CLI with the profiles that we want to use. We can add as many profiles into `~/.aws/config` file. An example of the config file is shown below : 

```yaml

[sso-session my-org-sso]
sso_start_url = https://my-org-aws-sso.awsapps.com/start
sso_region = us-east-1
sso_registration_scopes = sso:account:access

[profile staging-admin]
sso_session = my-org-sso
sso_account_id = 123456789012
sso_role_name = Admin

[profile staging-observer]
sso_session = my-org-sso
sso_account_id = 123456789012
sso_role_name = Observer
```
The above example is for AWS SSO. If you’re using AWS CLI with IAM credentials (I suggest not to use static IAM credentials), then you need to first configure the credential in `~/.aws/credentials` file like below : 

```yaml
[staging-admin]
aws_access_key_id=AKIAIOSFODNN7EXAMPLE
aws_secret_access_key=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY

[staging-observer]
aws_access_key_id=AKIAI44QH8DHBEXAMPLE
aws_secret_access_key=je7MtGbClwBF/2Zp9Utk/h3yCo8nvbEXAMPLEKEY
```

And then configure the profile in `~/.aws/config` file : 

```yaml
[profile staging-admin]
region = us-west-2

[profile staging-observer]
region = us-west-2
```

A detailed guide on how to configure AWS CLI with different profiles can be found [here](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-profiles.html).

## ZSH Functions with OhMyZsh

Since we’re using OhMyZsh, we can create a custom plugin and add the functions there.

In OhMyZsh, we can create a custom config inside `$ZSH_CUSTOM`. Any `*.zsh` file inside `$ZSH_CUSTOM` will be loaded automatically.

Lets create a file called `$ZSH_CUSTOM/aws.zsh` folder and add the below code : 

```bash
# function to list all the profiles in ~/.aws/config
function aws_profiles() {
  profiles=$(aws --no-cli-pager configure list-profiles 2> /dev/null)
  if [[ -z "$profiles" ]]; then
    echo "No AWS profiles found in '$HOME/.aws/config, check if ~/.aws/config exists and properly configured.'"
    return 1
  else
    echo $profiles
  fi
}

# function to set AWS profile, sso login and clear the profile
function asp() {
  available_profiles=$(aws_profiles)
  if [[ -z "$1" ]]; then
    unset AWS_DEFAULT_PROFILE AWS_PROFILE
    echo "Zero argument provided, AWS profile cleared."
    return
  fi
  
  echo "$available_profiles" | grep -qw "$1"
  if [[ $? -ne 0 ]]; then
    echo "Profile '$1' not configured in '$HOME/.aws/config'.\n"
    echo "Available profiles: \n$available_profiles\n"
    return 1
  else
    export AWS_DEFAULT_PROFILE="$1" AWS_PROFILE="$1"
  fi
}

# function to set AWS region and clear the region
function asr() {
  if [[ -z "$1" ]]; then
    unset AWS_DEFAULT_REGION AWS_REGION
    echo "No argument provided, cleared AWS region."
    return
  else
    export AWS_DEFAULT_REGION=$1 AWS_REGION=$1
  fi
}

# function to list all the profiles
function alp() {
  aws_profiles
}

# function to update the PS1 prompt with current AWS profile and region
function aws_ps1() {
  local profile_color="%{$(tput setaf 6)%}"  # Cyan color
  local region_color="%{$(tput setaf 2)%}"   # Green color
  local reset_color="%{$(tput sgr0)%}"      # Reset color
  echo -en "($profile_color$AWS_PROFILE$reset_color:$region_color$AWS_REGION$reset_color)"
}
```

Now, In order to use the `aws_ps1()` to change the prompt, we need to add the below code `~$HOME/.zshrc` file : 

```bash
PS1='$(aws_ps1)'$PS1
```

I usually prefer to have a default profile and region set when I open a new terminal. So, I’ll add the below code in my `$HOME/.zshrc` file : 

```bash
# set prod observer profile and us-west-2 as default
asp prod-observer
asr us-west-2
```

Once you save both the files and open a new terminal, you should see the below output : 

```bash
(prod-observer:us-west-2) ➜  ~
```

Let's list all the profiles : 
    
```bash
(prod-observer:us-west-2) ➜  ~ alp
staging-admin
staging-observer
```

Switch to staging-admin profile : 

```bash
(prod-observer:us-west-2) ➜  ~ asp staging-admin
(staging-admin:us-west-2) ➜  ~
```
Similarly, when the unknown profile is set: 


```bash
(staging-admin:us-west-2) ➜  ~ asp foobar

Profile 'foobar' not configured in '/Users/tanmay_bhat/.aws/config'.

Available profiles:
staging-admin
staging-observer

(staging-admin:us-west-2) ➜  ~
```

Using this, you don't need to give `--profile` and `--region` flags every time you run an AWS CLI command and no need to run `aws sts get-caller-identity` to check the current profile that you’re using before running a destructive command.