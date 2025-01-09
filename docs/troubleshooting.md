# Troubleshooting for production issues

This page enumerates some common user facing issues around GCSFuse and also
discusses potential solutions to the same.

### Generic Mounting Issue

Most of the common mount point issues are around permissions on both local mount point and the Cloud Storage bucket. It is highly recommended to retry with --foreground --log-severity=TRACE flags which would provide much more detailed logs to understand the errors better and possibly provide a solution.

### Mount successful but files are not visible

Try mounting the gcsfuse with `--implicit-dirs` flag. Read the [semantics](https://github.com/GoogleCloudPlatform/gcsfuse/blob/master/docs/semantics.md#files-and-directories) to know the reasoning.

### Mount failed with fusermount3 exit status 1

It comes when the bucket is already mounted in a folder, and we try to mount it again. You need to unmount first and then remount.

### version GLIBC_x.yz not found

GCSFuse should not be linking to glibc. Please either `export CGO_ENABLED=0` in your environment or prefix `CGO_ENABLED=0` to any <code>go build&#124;run&#124;test</code> commands that you're invoking.

### Mount get stuck with error: DefaultTokenSource: google: could not find default credentials

Run ```gcloud auth application-default login``` command to fetch default credentials to the VM. This will fetch the credentials to the following locations: <ol type="a"><li>For linux - $HOME/.config/gcloud/application_default_credentials.json</li><li>For windows - %APPDATA%/gcloud/application_default_credentials.json </li></ol>

### Input/Output Error

It’s a generic error, but the most probable culprit is the bucket not having the right permission for Cloud Storage FUSE to operate on. Ref - [here](https://stackoverflow.com/questions/36382704/gcsfuse-input-output-error)

### Generic NO_PUBKEY Error - while installing Cloud Storage FUSE on ubuntu 22.04

It happens while running - ```sudo apt-get update``` - working on installing Cloud Storage FUSE. You just have to add the pubkey you get in the error using the below command: ```sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys <PUBKEY> ``` And then try running ```sudo apt-get update```

### Cloud Storage FUSE fails with Docker container

Though not tested extensively, the [community](https://stackoverflow.com/questions/65715624/permission-denied-with-gcsfuse-in-unprivileged-ubuntu-based-docker-container) reports that Cloud Storage FUSE works only in privileged mode when used with Docker. There are [solutions](https://cloud.google.com/iam/docs/service-account-overview) which exist and claim to do so without privileged mode, but these are not tested by the Cloud Storage FUSE team

### SetUpBucket: OpenBucket: Bad credentials for bucket BUCKET_NAME: permission denied


Log may look similar to this ```daemonize.Run: readFromProcess: sub-process: mountWithArgs: mountWithStorageHandle: fs.NewServer: create file system: SetUpBucket: OpenBucket: Bad credentials for bucket BUCKET_NAME: permission denied```

Check the bucket name. Make sure it is within your project. Make sure the applied roles on the bucket  contain storage.objects.list permission. You can refer to them [here](https://cloud.google.com/storage/docs/access-control/iam-roles).

### SetUpBucket: OpenBucket: Unknown bucket BUCKET_NAME: no such file or directory

Log may look similar to this ```daemonize.Run: readFromProcess: sub-process: mountWithArgs: mountWithStorageHandle: fs.NewServer: create file system: SetUpBucket: OpenBucket: Unknown bucket BUCKET_NAME: no such file or directory```

Check the bucket name. Make sure the [service account](https://www.google.com/url?q=https://cloud.google.com/iam/docs/service-accounts&sa=D&source=docs&ust=1679992003850814&usg=AOvVaw3nJ6wNQK4FZdgm8gBTS82l) has permissions to access the files. It must at least have the permissions of the Storage Object Viewer role.

### mount: running fusermount: exit status 1 stderr: /bin/fusermount: fuse device not found, try 'modprobe fuse' first

Log may look similar to this - ```daemonize.Run: readFromProcess: sub-process: mountWithArgs: mountWithStorageHandle: Mount: mount: running fusermount: exit status 1 stderr: /bin/fusermount: fuse device not found, try 'modprobe fuse' first```

To run the container locally, add the --privilege flag to the docker run command: ```docker run --privileged  gcr.io/PROJECT/my-fs-app ``` <ul><li>You must create a local mount directory</li> <li>If you want all the logs from the mount process use the --foreground flag in combination with the mount command: ```gcsfuse --foreground --log-severity=TRACE $GCSFUSE_BUCKET $MNT_DIR ``` </li><li> Add --log-severity=TRACE for enabling debug logs</li></ul>

### Cloud Storage FUSE installation fails with an error at build time.

Only specific OS distributions are currently supported, learn more about [Installing Cloud Storage FUSE](https://github.com/GoogleCloudPlatform/gcsfuse/blob/master/docs/installing.md).

### Cloud Storage FUSE not mounting after reboot when entry is present in ```/etc/fstab``` with 1 or 2 as fsck order

Pass [_netdev option](https://github.com/GoogleCloudPlatform/gcsfuse/blob/master/docs/mounting.md#persisting-a-mount) in fstab entry (reference issue [here](https://github.com/GoogleCloudPlatform/gcsfuse/issues/1043)). With this option, mount will be attempted on reboot only when network is connected.

### Cloud Storage FUSE get stuck when using it to concurrently work with a large number of opened files (reference issue [here](https://github.com/GoogleCloudPlatform/gcsfuse/issues/1043))

This happens when gcsfuse is mounted with http1 client (default) and the application using gcsfuse tries to keep more than value of `--max-conns-per-host` number of files opened. You can try (a) Passing a value higher than the number of files you want to keep open to `--max-conns-per-host` flag. (b) Adding some timeout for http client connections using `--http-client-timeout` flag.

### Permission Denied error.

Please refer [here](https://cloud.google.com/storage/docs/gcsfuse-mount#authenticate_by_using_a_service_account) to know more about permissions
(e.g.  **Issue**:mkdir: cannot create directory ‘gcs/test’: Permission denied. User can check specific errors by enabling logs with --log-severity=TRACE flags.  
**Solution** - depending upon the use-case, you can choose one of the following options.
* If you are explicitly authenticating for a specific service account by providing say a key-file, then make sure that the service account has appropriate IAM role for the operation e.g. roles/storage.objectAdmin, roles/storage.objectUser
* If you are using the default service account i.e. not specifying a key-file, then ensure that
  * The VM's service account has got the required IAM roles for the operation e.g. roles/storage.objectUser to allow read-write access.
  * The VM's scope has been appropriately set. You can set the scope to storage-full to give the VM full-access to the cloud-storage buckets. For this:
    * Turn-off the instance
    * Change the VM's scope either by using the GCP console or by executing  `gcloud beta compute instances set-scopes INSTANCE_NAME --scopes=storage-full`
    * Start the instance

### Bad gateway error while installing/upgrading GCSFuse:
`Err: http://packages.cloud.google.com/apt gcsfuse-focal/main amd64 gcsfuse amd64 1.2.0`<br/>`502  Bad Gateway [IP: xxx.xxx.xx.xxx 80]`

This error is seen when the url used in /etc/apt/sources.list.d/gcsfuse.list file uses HTTP protocol instead of HTTPS protocol. Run the following commands to update /etc/apt/sources.list.d/gcsfuse.list file with the https:// url.<br/> <code>$ sudo rm /etc/apt/sources.list.d/gcsfuse.list</code> <br/> <code>$ export GCSFUSE_REPO=gcsfuse-$(lsb_release -c -s)</code> <br/> <code>$ echo "deb https://packages.cloud.google.com/apt $GCSFUSE_REPO main" &#124; sudo tee /etc/apt/sources.list.d/gcsfuse.list </code>

### Repository changed 'Origin' and 'Label' error while running apt-get update command:

<br/>`E: Repository 'http://packages.cloud.google.com/apt gcsfuse-focal InRelease' changed its 'Origin' value from 'gcsfuse-jessie' to 'namespaces/gcs-fuse-prod/repositories/gcsfuse-focal'`<br/>`E: Repository 'http://packages.cloud.google.com/apt gcsfuse-focal InRelease' changed its 'Label' value from 'gcsfuse-jessie' to 'namespaces/gcs-fuse-prod/repositories/gcsfuse-focal'`<br/>`N: This must be accepted explicitly before updates for this repository can be applied. See apt-secure(8) manpage for details. `

Use one of the following commands to upgrade to latest GCSFuse version<br/> `sudo apt-get update --allow-releaseinfo-change `<br/>OR<br/>`sudo apt update -y && sudo apt-get update`

### Unable to unmount or stop GCSFuse

Unable to unmount or stop GCSFuse due to an error message like:`fusermount: failed to unmount: Device or resource busy` or `umount: /path/to/mountpoint: target is busy`.</br>This typically indicates active processes are using files or directories within the GCSFuse mount.<br/>
Find the process ID of GCSFuse:<br/>`BUCKET=<Enter your bucket name>`</br>` MOUNT_POINT=<Enter your mount point>`</br>`PID=$(ps -aux &#124; grep "gcsfuse.*$BUCKET.*$MOUNT_POINT" &#124; grep -v grep &#124; tr -s ' ' &#124; cut -d' ' -f2)`</br>Kill the GCSFuse process:</br>`sudo kill -SIGKILL "$PID"`</br>Unmount GCSFuse</br>`fusermount -u $"MOUNT_POINT"`

### mount: exec: "fusermount": executable file not found in $PATH`

Unable to mount with the following error `daemonize.Run: readFromProcess: sub-process: Error while mounting gcsfuse: mountWithArgs: mountWithStorageHandle: Mount: mount: exec: "fusermount": executable file not found in $PATH`

Fuse package is not installed. It may throw this error. Run the following command to install fuse <br/> <code> sudo apt-get install fuse </code> for Debian/Ubuntu <br/> <code> sudo yum install fuse </code> for RHEL/CentOs/Rocky <br/>

### ls: reading directory \<mountpath>/\<path>:  Input/output error

Find out if you have any object(s) with name/prefix having `//` in it or starting with `/`, in the mounted GCS bucket (use `gsutil ls gs://<bucket>/<path>` to find out). If yes, move/rename such objects to name/prefix not having `//` or starting with `/` (e.g. for object/prefix `A//B`, use `gsutil -m mv -r gs://<bucket>/A//* gs://<bucket>/A/`), or delete it (use `gsutil -m rm -r A//`). Refer [semantics](semantics.md#unsupported-object-names) for more details.

### Experiencing hang while executing "ls" on a directory containing large number of files/directories.

By default `ls` does listing but sometimes additionally does `stat` for each list entry. Additional stat on each entry slows down the entire `ls` operation and may look like hang, but in reality is iterating one by one and slow. There are two ways to overcome this: <ul><li>**Get rid of `stat`:** Execute without coloring `ls --color=never` or `\ls` to remove the stat part of `ls`.</li> <li>**Improve the `stat` lookup time:** increase the metadata ([stat](https://cloud.google.com/storage/docs/gcsfuse-cache#stat-cache-overview) + [type](https://cloud.google.com/storage/docs/gcsfuse-cache#type-cache-overview)) cache ttl/capacity which is populated as part of listing and makes the `stat` lookup faster. Important to note here, `ls` will be faster from the first execution itself as cache is populated in listing phase, leading to quicker stat lookup for each entry.</li></ul>

### GCSFuse Errors on Google Cloud Console Metrics Dashboard: "CANCELLED ReadObject" & "NOT_FOUND GetObjectMetadata"

Both these errors are expected and part of GCSFuse standard operating procedure. More details [here](https://github.com/GoogleCloudPlatform/gcsfuse/discussions/2300).

### GCSFuse logs showing errors for StatObject NotFoundError 

`StatObject(\"<object_name>") (<time>ms): gcs.NotFoundError: storage: object doesn't exist"`.
This is an expected error. Please refer to **NOT_FOUND GetObjectMetadata** section [here](https://github.com/GoogleCloudPlatform/gcsfuse/discussions/2300#discussioncomment-10261838).

### Empty directory doesn't obey "--kernel-list-cache-ttl-secs" ttl.

In case a new file is added to the empty directory remotely, outside of the mount, the client will not be able to see the new file in listing even if ttl is expired.<br></br>Please don't use kernel-list-cache for multi node/mount-point file create scenario, this is recommended to be used only for read only workloads, e.g. for Serving and Training workloads. Or you need to create a new file/directory from the mount to see the newly added remote files.

### fuse: *fuseops.FallocateOp error: function not implemented or similar function not implemented errors

This is an expected error for file operations unsupported in FUSE file system (details [here](https://github.com/GoogleCloudPlatform/gcsfuse/discussions/2386#discussioncomment-10417635)).

### Error "transport endpoint is not connected"

It is possible customer is seeing the error "transport endpoint is not connected" because if the previous mount crashed or a mounted filesystem or device was abruptly disconnected or the mounting process failed unexpectedly, the system might not properly update its mount table. This leaves a stale entry referencing a resource that's no longer available.

**Solution:** Run the command `mount | grep "gcsfuse"`. If you find any entries, unmount the corresponding directory multiple times until all the entries cleared up and then try to remount the bucket.

**Additional troubleshooting steps:**

- Try to unmount and mount the mount-point using the command:
   `umount -f /<mount point>` && `mount /<mount point>`
- Try restarting/rebooting the VM Instance.

If it's running on GKE, the issue could be caused by an Out-of-Memory (OOM) error. Consider increasing the memory allocated to the GKE sidecar container. For more info refer [here](https://github.com/GoogleCloudPlatform/gcs-fuse-csi-driver/blob/main/docs/known-issues.md#implications-of-the-sidecar-container-design).