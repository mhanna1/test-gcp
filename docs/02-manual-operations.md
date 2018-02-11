# Manual Operations

To better understand the `Infrastructure as Code` (`IaC`) concept, we will first define the problem we are facing and deal with it with manually to get our hands dirty and see how things work overall.

## Intro

Imagine you have developed a new cool application called [raddit](https://github.com/Artemmkin/raddit).

You want to run your application on a dedicated server and make it available to the Internet users.

You heard about the `public cloud` thing, which allows you to provision compute resources and pay only for what you use. You believe it's a great way to test your idea of an application and see if people like it.

You've signed up for a free tier of [Google Cloud Platform](https://cloud.google.com/) (GCP) and are about to start deploying your application.

## Provision Compute Resources

First thing we will do is to provision a virtual machine (VM) inside GCP for running the application.

Use the following gcloud command in your terminal to launch a VM with Ubuntu 16.04 distro:

```bash    
$  gcloud compute instances create raddit-instance-2 \
      --image-family ubuntu-1604-lts \
      --image-project ubuntu-os-cloud \
      --boot-disk-size 200GB  \
      --machine-type n1-standard-1    

You should see similar to this

mark@L-Hanna-Mark:~$ gcloud compute instances create raddit-instance-2     --image-family ubuntu-1604-lts     --image-project ubuntu-os-cloud     --boot-disk-size 200GB     --machine-type n1-standard-1

Output:
Created [https://www.googleapis.com/compute/v1/projects/fluid-brook-194917/zones/us-east4-a/instances/raddit-instance-2].
NAME               ZONE        MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP    STATUS
raddit-instance-2  us-east4-a  n1-standard-1               10.150.0.2   35.188.225.79  RUNNING
```

## Create an SSH key pair

Generate an SSH key pair for future connections to the VM instances (run the command exactly as it is):

```bash
$ ssh-keygen -t rsa -f ~/.ssh/raddit-user -C raddit-user -P ""
```

Create an SSH public key for your project:

```bash
$ gcloud compute project-info add-metadata \
    --metadata ssh-keys="raddit-user:$(cat ~/.ssh/raddit-user.pub)"
    
Output:
Updated [https://www.googleapis.com/compute/v1/projects/fluid-brook-194917].    
```

You might want to skip the next two steps if you are not running a ssh agent.

Add the SSH private key to the ssh-agent:

```
$ ssh-add ~/.ssh/raddit-user
```

Verify that the key was added to the ssh-agent:

```bash
$ ssh-add -l
```

## Install Application Dependencies

To start the application, you need to first configure the environment for running it.

Connect to the started VM via SSH:

```bash
$ INSTANCE_IP=$(gcloud --format="value(networkInterfaces[0].accessConfigs[0].natIP)" compute instances describe raddit-instance-2)

mark@L-Hanna-Mark:~/.ssh$ INSTANCE_IP=$(gcloud --format="value(networkInterfaces[0].accessConfigs[0].natIP)" compute instances describe raddit-instance-2)
mark@L-Hanna-Mark:~/.ssh$ echo $INSTANCE_IP
35.188.225.79


$ ssh raddit-user@${INSTANCE_IP}

Output:
mark@L-Hanna-Mark:~/.ssh$ ssh raddit-user@${INSTANCE_IP} -i ~/.ssh/raddit-user
The authenticity of host '35.188.225.79 (35.188.225.79)' can't be established.
ECDSA key fingerprint is 83:b1:9c:69:06:be:c8:9a:0e:67:44:1f:98:e6:b3:66.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '35.188.225.79' (ECDSA) to the list of known hosts.
Welcome to Ubuntu 16.04.3 LTS (GNU/Linux 4.13.0-1008-gcp x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  Get cloud support with Ubuntu Advantage Cloud Guest:
    http://www.ubuntu.com/business/services/cloud

0 packages can be updated.
0 updates are security updates.



The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.
```




Install Ruby:

```bash
$ sudo apt-get update
$ sudo apt-get install -y ruby-full build-essential
```

Check the installed version of Ruby:

```bash
$ ruby -v
```

Install Bundler:

```bash
$ sudo gem install --no-rdoc --no-ri bundler
$ bundle version

Output:
raddit-user@raddit-instance-2:~$ sudo gem install --no-rdoc --no-ri bundler
Fetching: bundler-1.16.1.gem (100%)
Successfully installed bundler-1.16.1
1 gem installed
raddit-user@raddit-instance-2:~$ bundle version
Bundler version 1.16.1 (2017-12-21 commit 0034ef341)
```

Clone the [application repo](https://github.com/Artemmkin/raddit), but first make sure `git` is installed:
```bash
$ git version

Output:
raddit-user@raddit-instance-2:~$ git version
git version 2.7.4
```

At the time of writing the latest image of Ubuntu 16.04 which GCP provides has `git` preinstalled, so we can skip this step.

Clone the application repo into the home directory of `raddit-user` user:

```bash
$ git clone https://github.com/Artemmkin/raddit.git
```

Install application dependencies using Bundler:

```bash
$ cd ./raddit
$ bundle install
```

## Prepare Database

Install MongoDB which your application uses:

```bash
$ sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv EA312927
$ echo "deb http://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.2 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-3.2.list
$ sudo apt-get update
$ sudo apt-get install -y mongodb-org
```

Start MongoDB and enable autostart:

```bash
$ sudo systemctl start mongod
$ sudo systemctl enable mongod
```

Verify that MongoDB is running:

```bash
$ sudo systemctl status mongod

Output:
raddit-user@raddit-instance-2:~/raddit$ sudo systemctl enable mongod
Created symlink from /etc/systemd/system/multi-user.target.wants/mongod.service to /lib/systemd/system/mongod.service.
raddit-user@raddit-instance-2:~/raddit$
raddit-user@raddit-instance-2:~/raddit$
raddit-user@raddit-instance-2:~/raddit$ sudo systemctl status mongod
● mongod.service - High-performance, schema-free document-oriented database
   Loaded: loaded (/lib/systemd/system/mongod.service; enabled; vendor preset: enabled)
   Active: active (running) since Sun 2018-02-11 19:36:04 UTC; 30s ago
     Docs: https://docs.mongodb.org/manual
 Main PID: 19419 (mongod)
   CGroup: /system.slice/mongod.service
           └─19419 /usr/bin/mongod --quiet --config /etc/mongod.conf

Feb 11 19:36:04 raddit-instance-2 systemd[1]: Started High-performance, schema-free document-oriented database.
```

## Start the Application

Download a systemd unit file for starting the application from a gist:

```bash
$ wget https://gist.githubusercontent.com/Artemmkin/ce82397cfc69d912df9cd648a8d69bec/raw/7193a36c9661c6b90e7e482d256865f085a853f2/raddit.service
```

Move it to the systemd directory

```bash
$ sudo mv raddit.service /etc/systemd/system/raddit.service
```

Now start the application and enable autostart:

```bash
$ sudo systemctl start raddit
$ sudo systemctl enable raddit

Output:
raddit-user@raddit-instance-2:~/raddit$ sudo systemctl start raddit
raddit-user@raddit-instance-2:~/raddit$ sudo systemctl enable raddit
Created symlink from /etc/systemd/system/multi-user.target.wants/raddit.service to /etc/systemd/system/raddit.service.
```

Verify that it's running:

```bash
$ sudo systemctl status raddit
```

## Access the Application

Open a firewall port the application is listening on (note that the following command should be run on your local machine):

```bash
$ gcloud compute firewall-rules create allow-raddit-tcp-9292 \
    --network default \
    --action allow \
    --direction ingress \
    --rules tcp:9292 \
    --source-ranges 0.0.0.0/0
    
Output:
 gcloud compute firewall-rules create allow-raddit-tcp-9292 \
>     --network default \
>     --action allow \
>     --direction ingress \
>     --rules tcp:9292 \
>     --source-ranges 0.0.0.0/0
Creating firewall...\Created [https://www.googleapis.com/compute/v1/projects/fluid-brook-194917/global/firewalls/allow-raddit-tcp-9292].
Creating firewall...done.
NAME                   NETWORK  DIRECTION  PRIORITY  ALLOW     DENY
allow-raddit-tcp-9292  default  INGRESS    1000      tcp:9292

```

Get the public IP of the VM:

```bash
$ gcloud --format="value(networkInterfaces[0].accessConfigs[0].natIP)" compute instances describe raddit-instance-2
```

Now open your browser and try to reach the application at the public IP and port 9292.

```
NOTE
To learn how to host an image in github and to include the image in your documentation see the youtube video at ... https://www.youtube.com/watch?v=nvPOUdz5PL4    Slick Trick
```


For example, I put in my browser the following URL http://35.186.255.79:9292, but note that you'll have your own IP address.

![2018-02-11 15_01_22-raddit __on a google cloud platform posts](https://user-images.githubusercontent.com/12055220/36077806-31dd77fc-0f3d-11e8-96e4-8fc1d6e90e5a.jpg)


## Conclusion

Congrats! You've just deployed your application. It is running on a dedicated set of compute resources in the cloud and is accessible by a public IP. Now Internet users can enjoy using your application.

Now that you've got the idea of what sort of steps you have to take to deploy your code from your local machine to a virtual server running in the cloud, let's see how we can do it more efficiently.

Destroy the current VM and move to the next step:

```bash
$ gcloud compute instances delete raddit-instance-2
```

Next: [Scripts](03-scripts.md)
