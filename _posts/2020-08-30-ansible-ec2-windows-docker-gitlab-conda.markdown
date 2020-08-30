---
layout: post
title: "Ansible: using a Windows server EC2 VM to host a Gitlab-CI runner with docker-windows executor to build conda packages"
date: 2020-08-30 10:44:30 +0100
comments: true
author: Tom Schoonjans
categories: [python, conda-forge, anaconda, ec2, windows, ansible, docker, gitlab]
---


In this blogpost I will cover how to use Ansible to do all of the following in a single playbook:

1. [Create an EC2 instance using a Windows Server 2019 image](#create-an-ec2-instance-using-a-windows-server-2019-image)
2. [Create and attach an EBS volume to the instance](#create-and-attach-an-ebs-volume-to-the-instance)
3. [Initialize, partition and format the EBS volume](#initialize-partition-and-format-the-ebs-volume)
4. [Configure the Docker daemon](#configure-the-docker-daemon)
5. [Build our Docker images: vsbuildtools2019, miniconda3, miniforge3](#build-our-docker-images-vsbuildtools2019-miniconda3-miniforge3)
6. [Install and launch a Gitlab-CI runner with docker-windows executor](#install-and-launch-a-gitlab-ci-runner-with-docker-windows-executor)

I will finish this post with an [example](#configure-your-gitlab-repository-ci-to-use-these-images) of how you can make use of this runner from within a Gitlab-CI configuration file. At the very end you will find a [Gist](#the-gist-of-it) with the Ansible playbook and the associated files.

But before going into details, first a bit of background as to why I had to do all of this.

## Background

[Our team at the Franklin](https://www.rfi.ac.uk/science-themes/artificial-intelligence/) is currently developing the [RFI-File-Monitor](https://github.com/rosalindfranklininstitute/rfi-file-monitor), which will be used to archive, process and catalogue the data that is collected at our institute, according to a user-defined pipeline of operations that is applied to every file that occurs in an observed directory. Written in Python, development is split over two repositories: the core GUI and generic operations are available through a [public Github repo](https://github.com/rosalindfranklininstitute/rfi-file-monitor), while the operations that are specific to the Franklin (the _extensions_) are being developed in a private Gitlab repository.

To ensure that the Monitor can be easily installed on our instrument control machines, most of which are expected to run Windows, we need installers that will easily install this package and all of its dependencies. To create these installers, we use [conda-constructor](https://github.com/conda/constructor), which makes this task (relatively) easy, as it fully supports the Anaconda packaging system, on which we rely for the development of the Monitor.

As it is cumbersome to run the constructor script manually whenever a new release is made, a CI/CD pipeline was configured in the _extensions_ Gitlab repo to produce them automatically, and upload them to an S3 bucket. This blogpost explains in detail how the Gitlab-CI runner was set up to create the installers for the Windows platform using Ansible.

## Create an EC2 instance using a Windows Server 2019 image

Amazon offers [hundreds of images](https://aws.amazon.com/marketplace/search/results?searchTerms=ami) for different versions of Windows Server. In our case however, we are constrained by the [version requirements of the docker-windows executor of the Gitlab-CI runner](https://docs.gitlab.com/runner/executors/docker.html#supported-windows-versions), and by the fact that we are going to be using containers to get this up and running, so we end up with searching the image library with:

{% highlight yaml %}
{% raw %}
- name: List available Windows images
  ec2_ami_info:
    aws_access_key: "{{ aws_access_key }}"
    aws_secret_key: "{{ aws_secret_key }}"
    region: "{{ region }}"
    filters:
      name: "*2019*Core*Containers*"
      platform: windows
  register: win_ec2_windows_images
- name: Print last image
  debug:
    msg: "Image: {{ (win_ec2_windows_images.images | last).name }}"
{% endraw %}
{% endhighlight %}

In this case, we are interested in the last image in the returned list, as it corresponds to the [most recent build](https://aws.amazon.com/marketplace/pp/B07R3V4X23?qid=1598792600098&sr=0-2&ref_=srh_res_product_title).

Next, we need to create a security group for the instance that will allow us to provision it with WinRM (port 5986) and use RDP for an interactive session (port 3389), which is very useful for debugging:

{% highlight yaml %}
{% raw %}
- name: Create security group for win ec2 instance(s)
  ec2_group:
    name: '{{ security_group_name }}'
    description: "Rules for gitlab-ci-runner Windows VM"
    aws_access_key: "{{ aws_access_key }}"
    aws_secret_key: "{{ aws_secret_key }}"
    region: "{{ region }}"
    state: present
    rules: 
      - proto: tcp
        from_port: 3389
        to_port: 3389
        cidr_ip: 0.0.0.0/0
      - proto: tcp
        from_port: 5986
        to_port: 5986
        cidr_ip: 0.0.0.0/0
  register: win_ec2_security_group_result
{% endraw %}
{% endhighlight %}


An SSH public key is required, and must be installed on the VM. Do keep in mind that this key needs to be in the PEM format, which is not the default! I ended up using:

```shell
ssh-keygen -P "" -t rsa -b 4096 -m pem -f id_rsa_ec2
```

Import the keypair with:

```yaml
{% raw %}
- name: Import keypair
  ec2_key:
    name: "{{ key_name }}"
    key_material: "{{ lookup('file', ssh_public_key) }}"
    aws_access_key: "{{ aws_access_key }}"
    aws_secret_key: "{{ aws_secret_key }}"
    region: "{{ region }}"
    state: present
{% endraw %}
```

With this done, one can now spin up the EC2 instance:

```yaml
{% raw %}
- name: Create win ec2 instance
  ec2:
    instance_type: '{{ flavor }}'
    image: '{{ (win_ec2_windows_images.images | last).image_id }}'
    group_id: '{{ win_ec2_security_group_result.group_id }}'
    key_name: '{{ key_name }}'
    user_data: '{{lookup("file", "win_ec2_user_data")}}'
    exact_count: 1
    count_tag:
      Name: gitlab-ci-runner
    instance_tags:
      Name: gitlab-ci-runner
    wait: yes
    aws_access_key: "{{ aws_access_key }}"
    aws_secret_key: "{{ aws_secret_key }}"
    region: "{{ region }}"
  register: win_ec2_instance
- name: Print EC2 instance results
  debug:
    msg: "Image results: {{ win_ec2_instance }}"
{% endraw %}
```

Important here is the use of `exact_count`, `count_tag` and `instance_tags` to ensure that only one VM will be created, and that this will remain so when the Ansible script is run again! We will also be installing a small file with `user_data` to ensure it can be provisioned with Ansible later.

This instance comes with Windows Server installed on a drive of 30 GB, with about 18 GB left as free space. I initially thought that this would be enough to generate and store the Docker images, but this turned out to be false: the Docker image with a minimal installation of the Visual Studio build tools is huge!

## Create and attach an EBS volume to the instance.

The solution was to attach another disk that is sufficiently large to hold the Docker images. I used 50 GB, which did the trick:

```yaml
{% raw %}
- name: Attach EBS 50 GB
  ec2_vol:
    instance: '{{ win_ec2_instance.tagged_instances[0].id }}'
    volume_size: 50
    aws_access_key: "{{ aws_access_key }}"
    aws_secret_key: "{{ aws_secret_key }}"
    region: "{{ region }}"
    device_name: /dev/xvdg
    delete_on_termination: yes
{% endraw %}
```

At this point, we need to wait until the instance is ready to accept WinRM connections, and afterwards obtain the initial password that was given to the Administrator user on the VM. We will use this password to provision the VM, but it may also be used to log in via RDP:

```yaml
{% raw %}
- name: Wait for instance to listen on winrm https port
  wait_for:
    state: started
    host: '{{ win_ec2_instance.tagged_instances[0].public_ip }}'
    port: 5986
    delay: 5
    timeout: 360
- name: Obtain initial passwords for win ec2 instance
  ec2_win_password:
    instance_id: '{{ win_ec2_instance.tagged_instances[0].id }}'
    key_file: "{{ ssh_private_key }}"
    wait: yes
    aws_access_key: "{{ aws_access_key }}"
    aws_secret_key: "{{ aws_secret_key }}"
    region: "{{ region }}"
  register: win_ec2_password
- name: Print admin password
  debug:
    msg: "Admin password: {{ win_ec2_password.win_password }}"
{% endraw %}
```

Until now, we have been using localhost to execute these commands, but now we have to switch to the EC2 instance, so we need to start a new play after creating a new host for it to use:

```yaml
{% raw %}
- name: Store floating ip internally
  add_host:
    name: vm-ip
    ansible_host: '{{ win_ec2_instance.tagged_instances[0].public_ip }}'
    ansible_port: 5986
    ansible_user: Administrator
    ansible_password: '{{ win_ec2_password.win_password }}'
    ansible_winrm_server_cert_validation: ignore
    ansible_connection: 'winrm'
{% endraw %}
```

## Initialize, partition and format the EBS volume.

With the new play running, we can now hook up the EBS volume to the D: drive:

```yaml
{% raw %}
- name: Get disk facts
  win_disk_facts:
- name: Output disk facts
  debug:
    var: ansible_facts.disks
- name: Init, partition and format EBS
  block:
    # replace with win_initialize_disk in Ansible 2.10
    - name: Initialize disk
      win_command: powershell.exe -
      args:
        stdin: Initialize-Disk -Number 1
    - name: Partition EBS
      win_partition:
        drive_letter: D
        partition_size: -1
        disk_number: 1
    - name: Format EBS
      win_format:
        drive_letter: D
        file_system: NTFS
        full: no
  when: ansible_facts.disks[1].partition_count == 0
{% endraw %}
```

This will initialize the disk, partition it using all available space, and format the drive with NTFS.
With Ansible 2.9 we need to initialize the disk using a powershell command, as a dedicated module will only be available in 2.10. Since this command is not idempotent, I put these three actions into a block that is invoked only when the disk has not been partitioned.

## Configure the Docker daemon

The Docker daemon is configured to use the C: drive for storing caches, images etc by default. There is also a 20 GB limit on building and running images, which is rather close to the expected size of the images we need to build, so it is best to increase this:

```yaml
{% raw %}
- name: Create Docker Cache folder
  win_file:
    path: D:\DockerCache
    state: directory
- name: Install vim
  win_chocolatey:
    name: vim
    state: present
- name: Copy Docker daemon config file
  win_copy:
    src: daemon.json
    dest: C:\ProgramData\Docker\config\daemon.json
  register: daemon_json_copied
- name: Restart Docker
  win_service:
    name: docker
    state: restarted
  when: daemon_json_copied.changed
{% endraw %}
```

with daemon.json:

```json
{
    "storage-opts": [
    	"size=50GB"
    ],
    "data-root": "D:\\DockerCache"
}
```

Since we updated the Docker configuration, we need to restart it to ensure the changes are picked up. I am also install vim, using chocolatey, to help me with debugging via RDP.

## Build our Docker images: vsbuildtools2019, miniconda3, miniforge3

With Docker using the D: drive to store its data, we can now start building our Docker images. Since we plan on building binary Python modules, conda-build will need the Visual Studio Build Tools and its cl.exe compiler for C and C++ code, as well as the Windows SDK with its headers. I wrote three Dockerfiles:

# Dockerfile.vsbuildtools

A Dockerfile with the required command-line tools and headers from Visual Studio Build Tools

```docker
# escape=`

# Use the latest Windows Server Core image with .NET Framework 4.8.
FROM mcr.microsoft.com/dotnet/framework/sdk:4.8-windowsservercore-ltsc2019

# Restore the default Windows shell for correct batch processing.
SHELL ["cmd", "/S", "/C"]

# Download the Build Tools bootstrapper.
ADD https://aka.ms/vs/16/release/vs_buildtools.exe C:\TEMP\vs_buildtools.exe

# Install Build Tools with the Microsoft.VisualStudio.Workload.VCTools workload
RUN C:\TEMP\vs_buildtools.exe --quiet --wait --norestart --nocache `
    --installPath C:\BuildTools `
    --add Microsoft.VisualStudio.Workload.VCTools `
    --add Microsoft.VisualStudio.Component.VC.Tools.x86.x64 `
    --add Microsoft.VisualStudio.Component.Windows10SDK.18362 `
    --remove Microsoft.VisualStudio.Component.VC.CMake.Project `
    --remove Microsoft.VisualStudio.Component.VC.Llvm.Clang `
    --remove Microsoft.VisualStudio.Component.VC.Llvm.ClangToolset `
    --remove Microsoft.VisualStudio.ComponentGroup.NativeDesktop.Llvm.Clang `
    --locale en-US `
 || IF "%ERRORLEVEL%"=="3010" EXIT 0


# Define the entry point for the docker container.
# This entry point starts the developer command prompt and launches the PowerShell shell.
ENTRYPOINT ["C:\\BuildTools\\Common7\\Tools\\VsDevCmd.bat", "&&", "powershell.exe", "-NoLogo", "-ExecutionPolicy", "Bypass"]
```

# Dockerfile.miniconda

This Dockerfile extends the previous one with a miniconda installation, which currently comes with Python 3.8, and installs the conda-build package into it.

```docker
# escape=`

FROM buildtools2019:latest

# Restore the default Windows shell for correct batch processing.
SHELL ["cmd", "/S", "/C"]

# Download the Miniconda installer
ADD https://repo.anaconda.com/miniconda/Miniconda3-latest-Windows-x86_64.exe C:\TEMP\miniconda.exe

# Run the installer
RUN C:\TEMP\miniconda.exe /InstallationType=AllUsers `
    /AddToPath=1 `
    /RegisterPython=1 `
    /S `
    /D=C:\Miniconda

RUN conda update --all -y
RUN conda install -y conda-build

ENTRYPOINT ["C:\\BuildTools\\VC\\Auxiliary\\Build\\vcvars64.bat", "&&", "powershell.exe", "-NoLogo", "-ExecutionPolicy", "Bypass"]
```

# Dockerfile.miniforge

The last Dockerfile also extends the vsbuildtools2019 image with a miniconda installer, but updated all packages with their latest conda-forge counterparts, and made this channel default for all subsequent conda invocations.

```docker
# escape=`

FROM buildtools2019:latest

# Restore the default Windows shell for correct batch processing.
SHELL ["cmd", "/S", "/C"]

# Download the Miniconda installer
ADD https://repo.anaconda.com/miniconda/Miniconda3-latest-Windows-x86_64.exe C:\TEMP\miniconda.exe

# Run the installer
RUN C:\TEMP\miniconda.exe /InstallationType=AllUsers `
    /AddToPath=1 `
    /RegisterPython=1 `
    /S `
    /D=C:\Miniconda

RUN conda config --prepend channels conda-forge

RUN conda update --all -y
RUN conda install -y conda-build

ENTRYPOINT ["C:\\BuildTools\\VC\\Auxiliary\\Build\\vcvars64.bat", "&&", "powershell.exe", "-NoLogo", "-ExecutionPolicy", "Bypass"]
```

These three images are built using the `win_command` Ansible module, as the Docker Ansible module is unfortunately not supported on Windows. However, the docker commands are idempotent, which makes life easy:

```yaml
{% raw %}
- name: Create BuildTools folder
  win_file:
    path: C:\BuildTools
    state: directory
- name: Copy our Dockerfiles
  win_copy:
    src: 'Dockerfile.{{ item }}'
    dest: C:\BuildTools\
  loop:
    - vsbuildtools
    - miniconda
    - miniforge
- name: Build vsbuildtools Docker image
  win_command: docker build -t buildtools2019:latest -m 2GB -f Dockerfile.vsbuildtools .
  args:
    chdir: C:\BuildTools
- name: Build miniconda3 Docker image
  win_command: docker build -t miniconda -t miniconda3 -m 2GB -f Dockerfile.miniconda .
  args:
    chdir: C:\BuildTools
- name: Build miniforge3 Docker image
  win_command: docker build -t miniforge -t miniforge3 -m 2GB -f Dockerfile.miniforge .
  args:
    chdir: C:\BuildTools
{% endraw %}
```

## Install and launch a Gitlab-CI runner with docker-windows executor

While you may think that this would be the hardest part to get right, it turned out to be the easiest, as I could make use of the excellent [`riemers.gitlab-runner`](https://github.com/riemers/ansible-gitlab-runner) Ansible role, which I have used before successfully to install Gitlab-CI runners on an Openstack Linux VM. This role will install the runner, configure it and will register itself with the Gitlab instance whose repos will be making use of it:

```yaml
{% raw %}
- role: riemers.gitlab-runner
  # keep this until https://gitlab.com/gitlab-org/gitlab/-/issues/239013 is fixed
  gitlab_runner_wanted_version: 13.2.2
  gitlab_runner_registration_token: "{{ gitlab_registration_token }}"
  gitlab_runner_coordinator_url: "{{ gitlab_instance }}"
  gitlab_runner_runners:
    - name: 'GitLab Runner Docker Windows'
      executor: docker-windows
      docker_image: 'miniconda3'
      tags:
        - windows
      docker_volumes:
        - "C:\\cache"
      extra_configs:
        runners.docker:
          memory: 2048m
          pull_policy: never
          allowed_images:
            - miniconda
            - miniconda3
            - miniforge
            - miniforge3
            - buildtools2019
{% endraw %}
```

## Configure your Gitlab repository CI to use these images

The following example `.gitlab-ci.yml` demonstrates how to make use of these images. It assumes that your repo contains a Python package called `my-cool-package`, and with folders called `conda-build` and `conda-constructor` that contain the conda and constructor recipes (meta.yaml and construct.yaml).

```yaml
stages:
  - build
  - deploy

variables:
  AWS_DEFAULT_REGION: eu-west-2
  BUCKET_NAME: my-cool-bucket
  VERSION: 0.1.0

windows:build:
  stage: build
  image: miniforge3
  tags:
    - windows
  script:
    - conda build --python 3.8 conda-build
    - Copy-Item -Path C:\Miniconda\conda-bld\win-64\my-cool-package* -Destination .
  artifacts:
    paths:
      - my-cool-package*

windows:deploy:
  stage: deploy
  only:
    - tags
  image: miniconda3
  tags:
    - windows
  dependencies:
    - windows:build
  script:
    - New-Item -ItemType Directory -Path C:\Miniconda\conda-bld\win-64
    - New-Item -ItemType Directory -Path C:\Miniconda\conda-bld\noarch
    - Copy-Item -Path my-cool-package* -Destination C:\Miniconda\conda-bld\win-64\
    - conda index C:\Miniconda\conda-bld
    - conda install -y constructor
    - constructor conda-constructor
    - conda install -y -c conda-forge awscli
    - aws s3 cp my-cool-package-${VERSION}-Windows-x86_64.exe s3://${BUCKET_NAME}/${VERSION}/my-cool-package-${VERSION}-Windows-x86_64.exe
```

I use different images here in both steps: miniforge3 is used to build the conda package as I prefer to use conda-forge packages for this (I maintain many of the conda-forge feedstocks that are in the RFI-File-Monitor dependency stack). However, conda-constructor doesn't appear to work with conda-forge's conda-build, and requires the one from the defaults channel, so I have to use miniconda3 instead, even though the installer may include packages from any channel you want.

## The gist of it

And here are all the files in a gist... Let me know if you have questions!

{% gist f578f5b4aacf0cdbf03ea7b9cb06c5bd %}
