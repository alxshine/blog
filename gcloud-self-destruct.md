# Self destructing instances in the google cloud

So for my current project I needed instances in the google cloud that ran some experiments and then self destructed.
These experiments could take a while, so instead of having to regularly check on the cloud instances, I wanted them to upload the results to some cloud storage and then self-destruct.
The idea behind the self-destruct is that the instances wouldn't be running after being needed and incurring unnecessary costs.

## Previous work
Luckily, somebody already figured out a way how and shared it in this [Medium article](https://medium.com/google-cloud/how-to-make-a-self-destructing-vm-on-google-cloud-platform-b99883745b62).
By using the gcloud command line tools the instance can delete itself using the instance name and region.
The instance name *is actually the hostname*, which is super cool in this case, because we can just query that from the instance.
Finding the region is a bit trickier, but the author of the article already provided a curl snippet that queries it using the gcloud REST API:
```bash
$(curl -H Metadata-Flavor:Google http://metadata.google.internal/computeMetadata/v1/instance/zone -s | cut -d/ -f4)
```

## The problem
This command works well on full gcloud instances, but because my experiments all run in a docker container for portability (more on that maybe in another entry), I was using the container-OS image which doesn't have the gcloud SDK pre-installed.
Also it seems that for fully fledged OS instances a gcloud SDK config is copied to the device, which allows it to authenticate it as the service-account that was used to create the instance in the first place.

So, we have two open questions:

- how do we get the gcloud SDK onto the instance?
- how do we get the credentials onto the instance?

## Getting the gcloud SDK
As I said earlier the instances were running the specialized and lightweight container-OS, which basically has nothing but docker installed.
However, the gcloud SDK is actually available as a docker image and can be pulled from the Google Container Registry.
This means we don't have to install anything, and can just run the `gcloud` command from the container.

Cool, so one problem solved.

## Authenticating the gcloud SDK command
The Google cloud interface is not really meant for small projects.
It only has one interface, and that has to be good enough for the largest possible customers.
This means that for my research I have to add all the bells and whistles you would expect for enterprise grade access management and authentication.
This meant that giving the service account the correct permissions to actually delete an instance in the compute engine was a bit of a pain.
The command is actually nice enough to tell you what permissions the account would need, however you can't add permissions directly.
Instead you have to add roles, which themselves contain permissions.
Sigh...

After adding the correct permissions, the instance still needs to authenticate as the service account it claims to be.
For this it needs the private key of the service account.
You can download these from the [service accounts page under IAM&admin in the gcloud console](https://console.cloud.google.com/iam-admin/serviceaccounts).

We have the datasets and weights we need stored on gcloud disks that are attached as read only to the instances.
This way we don't have to duplicate them for each instance.
Similarly, we created another disk for storing the credentials, and copied the key file onto the disk.

Before you can actually issue commands using the gcloud SDK though, you still need to configure values like the account, project, and region.
I did this manually by SSHing into an instance and running `gcloud init`, storing the result on the credentials disk, which was mounted as read-write for this maintenance job.
This way we can copy a raw gcloud config into the home folder of our user for each instance, and issue the self destruct command.

## Bringing it all together
Our first variant stayed connected to the newly created instance and executed a script on it via the gcloud SDK.
However, because the instance destroys itself the script crashes, and handling that (intended) crash seemed like a headache.
Instead, we built a script to

1. do all our setup stuff
2. run the experiments
3. self-destruct the instance

and pass that as a [startup script](https://cloud.google.com/compute/docs/startupscript).
Startup scripts are are run automatically on instance creation, and can be used for setup and starting services.
Ours did setup work like copying the gcloud config, getting the region, and mounting the disks.

After the initial configuration the startup script runs the docker container containing our experiments.
The datasets, weights, and gcloud credentials are all mounted into the container using [bind mounts](https://docs.docker.com/storage/bind-mounts/), which I can highly recommend.
The credentials are used to upload the experiment results from the docker container directly into a Google cloud storage bucket, from where we can download them later.
This is what really enables this "fire and forget" experiment style.

After that the actual *magic* happens.
We run the gcloud-sdk docker, using the following command:
```bash
docker run --rm \
  --mount type=bind,src=/mnt/disks/credentials/,dst=/credentials \
  --mount type=bind,src=$TARGET_HOME/gcloud_config,dst=/config \
  -e CLOUDSDK_CONFIG=/config \
  gcr.io/google.com/cloudsdktool/cloud-sdk \
  $DELETE_COMMAND
```

As you can see we not only mount the credentials, but also the gcloud config from the current users home folder.
We then tell the gcloud SDK where to find the configuration using the `CLOUDSDK_CONFIG` environment variable.
The `DELETE_COMMAND` is a plain old *bash* variable, using the curl snippet for finding the current region:
```bash
DELETE_COMMAND="gcloud compute instances delete $(hostname) -q \
--zone $(curl -H Metadata-Flavor:Google http://metadata.google.internal/computeMetadata/v1/instance/zone -s | cut -d/ -f4)"
```

## Conclusion
This is how we used self-destructing gcloud instances to allow for easier experimentation.
We can now create all required instances immediately, and let them run in parallel, while keeping excess cost to a minimum.
Another cool insight gained from this is that looking at the cost spike in billing I can say that running our experiments costs us roughly 50 cents per run.
I think that's a reasonably low price to pay for science ;)
