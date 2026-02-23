```bash
sudo apt-get update
sudo apt-get install -y curl gnupg2
curl -fsSL https://packages.plakar.io/keys/plakar.gpg | sudo gpg --dearmor -o /usr/share/keyrings/plakar.gpg
echo "deb [signed-by=/usr/share/keyrings/plakar.gpg] https://packages.plakar.io/deb stable main" | sudo tee /etc/apt/sources.list.d/plakar.lis
```

```bash
sudo apt-get update
sudo apt-get install plakar
```

### Create a Kloset Store
Before we can back up any data, we need to define where the backup will go. In plakar terms, this storage location is called a `Kloset Store`. This is where Plakar keeps your backups. Think of it like a safe folder for snapshots.

```bash
ubuntu@ip-172-31-28-74:~$ echo $HOME
/home/ubuntu
ubuntu@ip-172-31-28-74:~$ ls -ltr
total 0
```

```bash
ubuntu@ip-172-31-28-74:~$ plakar at $HOME/backups create
```
plakar will then ask you to enter a passphrase, and repeat it to confirm.

**Be extra careful when choosing the passphrase: People with access to the Kloset Store and knowledge of the passphrase can read your backups.

By default plakar will enforce rules on your choice of passphrase to make sure it is complex enough to be secure. To add complexity, use a mixture of upper and lower case characters, numbers and symbols.

DO NOT LOSE OR FORGET THE PASSPHRASE: it is not stored anywhere and cannot be recovered in case of loss. A lost passphrase means the data within the repository can no longer be accessed or recovered.**

```bash
ubuntu@ip-172-31-28-74:~$ ls -ltr
total 4
drwx------ 5 ubuntu ubuntu 4096 Feb 23 14:42 backups
ubuntu@ip-172-31-28-74:~$
```

### Create your first backup
Now that we have created the Kloset Store where data will be stored, we can use it to create our first backup. plakar uses the `at` keyword to specify the Kloset Store to use.

To create a simple example backup, try running:
```bash
plakar at $HOME/backups backup $HOME/original
```

The output indicate the progress
```bash
repository passphrase:
4ad7b85d: OK ✓ /home/ubuntu/original/sample.txt
4ad7b85d: OK ✓ /home/ubuntu/original
4ad7b85d: OK ✓ /home/ubuntu
4ad7b85d: OK ✓ /home
4ad7b85d: OK ✓ /
info: backup: created unsigned snapshot 4ad7b85d of size 19 B in 38.641082ms (wrote 201 KiB)
```
* The output lists the short form of the snapshot ID. This is used to identify a particular snapshot and is also how you identify the snapshot to use for various plakar commands.

### List snapshots
You can verify that the backup exists with the ls command, which returns the backups in that Kloset Store:
```bash
plakar at $HOME/backups ls
repository passphrase:
2026-02-23T14:47:38Z   4ad7b85d      19 B        0s /home/ubuntu/original
```
The output lists the date of the last backup, the short UUID, the size of files backed-up, the time it took to create the backup and the source path of the backup.


### Verify integrity
It’s always a good idea to verify the integrity of your backups. You can do this with the `check` command. This will read back the data from the Kloset Store, decrypt it and verify its integrity by recomputing checksums.

```bash
$ plakar at /home/ubuntu/backups check 4ad7b85d
repository passphrase:
info: 4ad7b85d: ✓ /home/ubuntu/original
info: 4ad7b85d: ✓ /home/ubuntu/original/sample.txt
info: check: verification of 4ad7b85d:/home/ubuntu/original completed successfully
```

### Restore files from a backup
You can restore files from a backup using the `restore` command. In this case, we are restoring the snapshot we just created to another directory called `restored`.

```bash
$ plakar at /home/ubuntu/backups restore -to /home/ubuntu/restored 4ad7b85d
repository passphrase:
info: 4ad7b85d: OK ✓ /home/ubuntu/original
info: 4ad7b85d: OK ✓ /home/ubuntu/original/sample.txt
info: restore: restoration of 4ad7b85d:/home/ubuntu/original at /home/ubuntu/restored completed successfully
```

Verification of restoration:
```bash
tree restored/
restored/
└── original
    └── sample.txt

2 directories, 1 file
ubuntu@ip-172-31-28-74:~$ ls
backups  original  restored
```

### Plakar UI
```bash
ubuntu@ip-172-31-28-74:~$ plakar at /home/ubuntu/backups ui
repository passphrase:
failed to launch browser: xdg-open and browser fallback failed: exec: "xdg-open": executable file not found in $PATH
you can access the webUI at http://localhost:31912?plakar_token=b3d48e34-1890-4eb3-8fce-439e66263f10
launching webUI at http://localhost:31912?plakar_token=b3d48e34-1890-4eb3-8fce-439e66263f10
```
## Synchronize multiple copies

### Why create multiple copies?
Keeping multiple backup copies dramatically reduces the risk of total data loss by **turning a realistic single-site failure into an extremely unlikely event when data is replicated across independent locations**

### Login to install pre-built integrations
By default, `plakar` works without requiring you to create an account or log in. You can back up and restore your data with just a few commands, with no external services involved.

However, logging in unlocks optional features that improve usability and monitoring. Among these features, it adds the ability to install the pre-built integrations hosted on our infrastructure.

In this quickstart, we will use the S3 integration, which requires the integration to be installed first. Therefore, we need to log in.
```bash
$ plakar login -email <youremailaddress@example.com>
```

To check that you are now logged in, you can run:
```bash
$ plakar login -status
logged in
```

### Install the S3 integration
Run the following command to install the S3 integration:
```bash
$ plakar pkg add s3
```

You can list all installed integrations to confirm the S3 integration was installed successfully:
```bash
$ plakar pkg list
s3@v1.0.7
```

### Set up S3-compatible storage
#### Configure Plakar
To let `plakar` know about the S3 storage, we need to configure a new store. We will call this store `s3-backups`

```bash
$ plakar store add s3-backups location=s3://plakar-backups-230226.s3.ap-southeast-2.amazonaws.com/mybucket access_key=AKI<REDACTED>L5466 secret_access_key=8rwgJ<REDACTED>BCzE use_tls=false
```

This command creates a new store named s3-backups that points to the mybucket bucket on the MinIO server running at localhost:9000. It uses the access key and secret key provided above. The use_tls=false option is specified because we are connecting to a local server without TLS.

### Initialize the Kloset Store
For now, the Kloset Store points to a bucket that does not exist yet. We need to create it by initializing the store:
```bash
$ plakar at "@s3-backups" create
repository passphrase:
insecure password, try including more special characters or using a longer password
repository passphrase:
repository passphrase (confirm):
```


This command initializes the Kloset Store at the S3 location, creating the necessary bucket and structure to hold the backups.

Note the @ symbol before the store name. This is an alias, which indicates that we are referencing a Kloset Store from the configuration. Without the @, plakar would interpret s3-backups as a filesystem path.

### List snapshots
If you list the snapshots in this new store, you will see that it is currently empty:
```bash
$ plakar at "@s3-backups" ls
repository passphrase:
ubuntu@ip-172-31-28-74:~$
```

### Synchronize the local Kloset Store to S3
Now, let’s synchronize the local Kloset Store at `$HOME/backups` to the S3 Kloset Store we just created.

Run the following command:
```bash
$ plakar at $HOME/backups sync to "@s3-backups"
repository passphrase:
destination store passphrase:

info: Synchronizing snapshot 4ad7b85ded2777e619d5db85626022e9cca3f5c2c53be1e3984954eeb245ce9f from fs:///home/ubuntu/backups to s3://plakar-backups-230226.s3.ap-southeast-2.amazonaws.com/mybucket
info: Synchronization of 4ad7b85ded2777e619d5db85626022e9cca3f5c2c53be1e3984954eeb245ce9f finished
info: sync: synchronization from fs:///home/ubuntu/backups to s3://plakar-backups-230226.s3.ap-southeast-2.amazonaws.com/mybucket completed: 1 snapshots synchronized
```

The command transfers all the snapshots from the local Kloset Store to the S3 Kloset Store.

To verify that the synchronization was successful, you can list the snapshots in the S3 Kloset Store again:
```bash
$ plakar at "@s3-backups" ls
```