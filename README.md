---
tags: rclone, google sa, gist (public)
---
# Google Service Account for Rclone Setup - A Complete Tutorial
This is a manual tutorials to get into details to understand the entire process of creating Google SA (Service Accounts), and how to use those to do a rclone task with automatic switching when a condition (limit) is met.

THIS IS NOT A GUIDE TO SETUP RCLONE REMOTE. 

THIS IS NOT A GUIDE TO SETUP RCLONE REMOTE. 

THIS IS NOT A GUIDE TO SETUP RCLONE REMOTE. 

# Pre-requisite
**1. Create google group**
Create a group so that the SA can have the email with it.

Create google group https://groups.google.com/forum/?oldui=1#!creategroup

Remember this googlegroup email becase we will use it in `GROUP_NAME` in `sa-gen` script below. All SA account will have their suffix with the googlegroup email.

**2. Install google cloud sdk**
```
# Add the Cloud SDK distribution URI as a package source
echo "deb http://packages.cloud.google.com/apt cloud-sdk main" | sudo tee -a /etc/apt/sources.list.d/google-cloud-sdk.list

# Import the Google Cloud Platform public key
curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

# Update the package list and install the Cloud SDK
sudo apt-get update && sudo apt-get install google-cloud-sdk
```

**3. Initialize SDK**
```
gcloud init --console-only

# next select a project, gcloud sdk will use that credential by default
```

# Main
**1. Generate SA**
```
cd ~
git clone https://github.com/88lex/sa-gen
cd sa-gen
chmod +x sa-gen

# Create an output for the SA jsons
mkdir ~/sa
```

Modify `sa-gen` with the following. This will create just one project with a maximum of 100 SA.
Note: The values CANNOT have underscore, just dash
```
KEYS_DIR=/home/deluge23/sa
ORGANIZATION_ID="123456789101" # a 12 digits ID, check from IAM 
GROUP_NAME="MYGROUP@googlegroups.com"
PROJECT_BASE_NAME="turnip-sa-sync"
FIRST_PROJECT_NUM=1
LAST_PROJECT_NUM=2
SA_EMAIL_BASE_NAME="turnip-sagen"
FIRST_SA_NUM=1
NUM_SAS_PER_PROJECT=100
CYCLE_DELAY=0.1s                  # If issues with Google back end not recognizing SAs increase this number. Set = 0 for no delay
SECTION_DELAY=5s               # If issues with Google back end not recognizing SAs increase this number. Set = 0 for no delay
```

Read https://github.com/88lex/sa-gen for details for each field.

Run the script to start generating those SA.jsons.
```
./sa-gen
```

You should see a list of .jsons at `~/sa`
```
ls ~/sa
100.json  16.json  22.json  29.json  35.json  41.json  48.json  54.json  60.json  67.json  73.json  7.json   86.json  92.json  99.json
10.json   17.json  23.json  2.json   36.json  42.json  49.json  55.json  61.json  68.json  74.json  80.json  87.json  93.json  9.json
11.json   18.json  24.json  30.json  37.json  43.json  4.json   56.json  62.json  69.json  75.json  81.json  88.json  94.json  allmembers.csv
12.json   19.json  25.json  31.json  38.json  44.json  50.json  57.json  63.json  6.json   76.json  82.json  89.json  95.json  members.csv
13.json   1.json   26.json  32.json  39.json  45.json  51.json  58.json  64.json  70.json  77.json  83.json  8.json   96.json
14.json   20.json  27.json  33.json  3.json   46.json  52.json  59.json  65.json  71.json  78.json  84.json  90.json  97.json
15.json   21.json  28.json  34.json  40.json  47.json  53.json  5.json   66.json  72.json  79.json  85.json  91.json  98.json

```

**2. Create Team Drive**
Go to drive.google.com and create a Team Drive.

**3. Add SA to TeamDrive**
*Important: This step is important because otherwise those SA account generated above CANNOT access the team drive.*

Open `allmembers.csv` from `~/sa` to see a list of SA members with their group email and member email.

Add members (member email) to the TeamDrive using Add Members.
![](https://i.imgur.com/aVxfeWY.png)


**4. Create a new rClone remote for the TeamDrive**
```
rclone config ...

# After creating the remote, test connecting using the SA account.
# Only the SA account that you add to the Team Drive in Step 3 can access the team drive.
rclone lsd myteamremote: --drive-service-account-file=/home/deluge23/sa/1.json
```

**5. Setup Rclone Sync**
Read below.

# Setup Rclone Sync
There are two methods that I tested to execute rClone task (copy/move/sync) using those SA accounts. I've tried `sasync` but it is currently NOT able to use SA to sync to CRYPT remote. It always uses my main account instead of SA during those task. Syncing to regular DRIVE remote works with SA. **So only use `sasync` if your target is a regular DRIVE, not a CRYPT**

**Sanity Check**
Regardless of which method you chose, always open up your TeamDrive via drive.google.com for the first time during when rClone task is running and check who is the uploader. It setup properly it should be your SA email, not your main account!

Example of correct setup
![](https://i.imgur.com/S4GLEEG.png)


## Method 1: autorclone by Rhilips
This method works even if your destination is a [CRYPT].

### How It Works
It will monitor the rclone task output every 10 seconds via `rclone rc core/stats` and extract some relevant fields to determine whether a condition is met before switching to SA.

Example of `rclone rc core/stats` in JSON format:
```
# rclone rc core/stats
{
        "bytes": 79722449276,
        "checks": 321,
        "deletes": 0,
        "elapsedTime": 1253.847935281,
        "errors": 0,
        "fatalError": false,
        "renames": 0,
        "retryError": false,
        "speed": 63582233.47126536,
        "transferring": [
                {
                        "bytes": 55541160800,
                        "eta": 31,
                        "group": "global_stats",
                        "name": "Iron Man (2008)/Iron.Man.2008.UHD.BluRay.2160p.TrueHD.Atmos.7.1.HEVC.REMUX-FraMeSToR.mkv",
                        "percentage": 97,
                        "size": 56958243521,
                        "speed": 44527171.62684323,
                        "speedAvg": 44468415.93373678
                }
        ],
        "transfers": 4
}

```

It supports 4 conditions to switch to another SA account.

When a (or multiple) condition is met, it will force kill the existing rClone process, then wait 60s before starting another rClone instance using another SA.

The next SA chosen is random.

### Setup
```
cd ~
git clone https://github.com/Rhilip/AutoRclone/
cd AutoRclone
```

There are several things to modify,
**1. Change the sa account location:**
```
sa_json_folder = r'/home/deluge23/sa'  # 绝对目录，最后没有 '/'，路径中不要有空格
```

**2. Change the rclone task that you want to run.**
There is no need to specify `--max-transfer` option here because `autorclone` have several condition to auto matically switch over to the next random SA.
```
cmd_rclone = 'rclone copy local:/mnt/downloads/media-staging gcrypt-gdrive-team-media:/movies-test -vP --log-file /tmp/rclone.log --include-from /home/deluge23/rclone-out.txt --transfers 1 --checkers 2 --cutoff-mode=soft'
```

**3. Enable the condition(s) to switch SA account**
By default all rules (4 total) are disabled, so it won't switch SA account. Change it to `True` to turn on the rule. 

Example below will switch SA if:
- Uploaded more than 750GiB
- User rate limit error
- 0 bytes transferred
```
switch_sa_rules = {
    'up_than_750': True,  # 当前帐号已经传过750G
    'error_user_rate_limit': True,  # Rclone 直接提示rate limit错误
    'zero_transferred_between_check_interval': True,  # 100次检查间隔期间rclone传输的量为0
    'all_transfers_in_zero': False,  # 当前所有transfers传输size均为0
}
```

**4. Set how many rules to hit before switching SA**
By default meeting the condition of any one of the rule will trigger the switch. Increase the number to make it strict.
```
switch_sa_level = 1  # 需要满足的规则条数，数字越大切换条件越严格，一定小于下面True（即启用）的数量，即 1 - 4(max)
```

Example of SA switching if `switch_sa_rules` `up_than_750` is met.
```
...
08/17/2020 03:05:12 PM - INFO - <module> - Transfer Status - Upload: 697.3329766681418 GiB, Avg upspeed: 45.63032140195016 MiB/s, Transfered: 76.
08/17/2020 03:05:22 PM - INFO - <module> - Transfer Status - Upload: 697.7072991123423 GiB, Avg upspeed: 45.625559122140636 MiB/s, Transfered: 76.
08/17/2020 03:05:32 PM - INFO - <module> - Transfer Status - Upload: 698.0907891681418 GiB, Avg upspeed: 45.62138333621777 MiB/s, Transfered: 76.
08/17/2020 03:05:42 PM - INFO - <module> - Transfer Status - Upload: 698.5075737973675 GiB, Avg upspeed: 45.61939164491666 MiB/s, Transfered: 76.
08/17/2020 03:05:42 PM - INFO - <module> - Transfer Limit may hit (Switch Reason: Rule `up_than_750` hit, ), Try to Switch..........
08/17/2020 03:05:42 PM - INFO - force_kill_rclone_subproc_by_parent_pid - Get The Process information - pid: 26634, name: sh
08/17/2020 03:05:42 PM - INFO - force_kill_rclone_subproc_by_parent_pid - Force Killed rclone process which pid: 26635
08/17/2020 03:05:42 PM - INFO - <module> - Switch to next SA..........
08/17/2020 03:05:43 PM - INFO - <module> - Get SA information, file: /home/deluge23/sa/97.json , email: turnips-sagen97@turnips-sa-sync1.iam.gserviceaccount.com
08/17/2020 03:05:43 PM - INFO - <module> - Wait 60 seconds to full call rclone command: rclone copy local:/mnt/downloads/media-staging gcrypt-gdrive-team-media:/m
ovies-test -vP --log-file /tmp/rclone.log --include-from /home/deluge23/rclone-out.txt --transfers 1 --checkers 2 --cutoff-mode=soft --rc --drive-service-account-file /h
ome/deluge23/sa/97.json
08/17/2020 03:06:43 PM - INFO - <module> - Run Rclone command Success in pid 2675
08/17/2020 03:06:43 PM - INFO - <module> - Transfer Status - Upload: 2.015625 GiB, Avg upspeed: 38.15953547219405 MiB/s, Transfered: 0.
...
```

**Summary**
- SA account are chosen at random
- Logs are at `/tmp/autorclone.log`. 
    - Change the location at `script_log_file = r'/tmp/autorclone.log'`. 
    - Monitor the log by `tail -f /tmp/autorclone.log`
- 4 conditions to switch SA account. 



## Method 2: sasync
**!!! NOTE: USE ONLY IF YOUR DESTINATION IS A REGULAR [DRIVE] REMOTE, NOT A [CRYPT]!!!**

sasync is used to auto cycle a list of rclone task that you can define and executed by those SA (copy/move/sync)
```
cd ~
git clone https://github.com/88lex/sasync
cd sasync
```

Configure the sasync config
```
cp sasync.conf.default sasync.conf
```

An example of my sasync.conf
```
# sasync.conf
#!/usr/bin/env bash
# Double check that these flags point to your files/folders
SASDIR="/home/deluge23/sasync"              # Location of the sasync script
SETDIR="$SASDIR/sets"             # Location of your set files
JSDIR="/home/deluge23/sa"                   # Location of DIR with SA .json files. No trailing slash
MINJS=1                           # First json file number in your JSON directory (JSDIR)
MAXJS=100                        # Hughest/max json file number in your JSON directory (JSDIR)
JSCOUNT="$SASDIR/json.count"      # Location of json.count file (NOT the jsons themselves)
NEXTJS=1                          # Cycle json by NEXTJS. Default 1. Using 101 may help avoid api issues
FILTER="$SASDIR/filter"           # Location of your default filter file
IFS1=$' \t\n'                     # Field separator for set file. Use ',' or '|' if you do not use space

# Flags to control which checks and what cleaning is done. All = false works in standard case
CHECK_REMOTES=true                # Check if remotes are configured in rclone. Faster if set to false
CALC_SIZE=false                   # Runs check size ; MUST set to true if using DIFF_LIMIT
DIFF_LIMIT=0                      # If source/dest size < or = DIFFLIMIT % then skip pair (def=0, use integers)
FILE_COMPARE=false                # Runs hash check against files
CLEAN_DEST=false                  # Set to true if you want to clean the destination
CLEAN_SRC=false                   # Set to true if you want to clean the source
PRE_CLEAN_TDS=false               # Set to true to clean remotes before running rclone
EXIT_ON_BAD_PAIR=false            # Exit (true) or continue (false) if set pair is bad
SRC_LIMIT=                        # Daily limit in GB. Blank if none. To use, CALC_SIZE must be true
BAK_FILES=false                   # (true/false)Send files to backup dir rather than delete them
BAK_DIR=backup                    # Backup files sent to destination/BAK_FILE_DIR
MAKE_DESTDIR=false                # To make missing directories set to true.

# These flags are applied to all sets. Tweak them as you like
FLAGS="
  --fast-list
  -vP
  --tpslimit-burst=50
  --stats=10s
  --max-backlog=2000000
  --ignore-case
  --no-update-modtime
  --drive-chunk-size=256M
  --drive-use-trash=false
  --filter-from=$FILTER
  --track-renames
  --use-mmap
  --drive-server-side-across-configs=true
  --drive-stop-on-upload-limit
  "

```

Modify the set file which will list the rclone task.

Following is my set file to copy movies from local drive to the remote.
```
# use the sample template as starting point
cp sets/set.video.sample sets/media-staging.set
```

### Sets example
**Example 1: Copy movies from local drive to remote**
```
nano sets/media-staging.set

# Note: rclone_flags are optional and will override global flags
#synccopymove 1source        2destination   3rclone_flags
copy          local:/mnt/downloads/media-staging   gcrypt-gdrive-team-media:/movies   --include-from /home/deluge23/rclone-out.txt -P --max-transfer 750G --transfers 1 --checkers 2 --cutoff-mode=soft
#end-of-file
```

Note: 
You must first create a `local` remote in `rclone config` because the script only accept rclone remote as source/destination.

```
# Example of ~/.config/rclone.conf
...
[local]
type = local
```

After you have the config and task set, run it
```
# sasync will search for the .set in the ./sets folder, so you don't have to specify the relative folder location
./sasync -c sasync.conf media-staging.set
```

And voila!
```
 * Toy Story 4 (2019)/Toy…Atmos - KRaLiMaRKo.mkv:  3% /31.758G, 81.444M/s, 6m2Transferred:        138.201G / 651.823 GBytes, 21%, 85.265 MBytes/s, ETA 1h42m48s
Transferred:           20 / 84, 24%
Elapsed time:     27m39.7s
Transferring:
 * Toy Story 4 (2019)/Toy…Atmos - KRaLiMaRKo.mkv:  6% /31.758G, 87.633M/s, 5m4Transferred:        139.009G / 651.823 GBytes, 21%, 85.249 MBytes/s, ETA 1h42m39s
Transferred:           20 / 84, 24%
Elapsed time:     27m49.7s
Transferring:
 * Toy Story 4 (2019)/Toy…Atmos - KRaLiMaRKo.mkv:  9% /31.758G, 85.359M/s, 5m4Transferred:        139.937G / 651.823 GBytes, 21%, 85.308 MBytes/s, ETA 1h42m24s
Transferred:           20 / 84, 24%
Elapsed time:     27m59.7s
Transferring:
 * Toy Story 4 (2019)/Toy…Atmos - KRaLiMaRKo.mkv: 12% /31.758G, 90.275M/s, 5m1Transferred:        140.783G / 651.823 GBytes, 22%, 85.316 MBytes/s, ETA 1h42m13s
Transferred:           20 / 84, 24%
Elapsed time:      28m9.7s
Transferring:
 * Toy Story 4 (2019)/Toy…Atmos - KRaLiMaRKo.mkv: 14% /31.758G, 89.147M/s, 5m1Transferred:        141.642G / 651.823 GBytes, 22%, 85.331 MBytes/s, ETA 1h42m2s
Transferred:           20 / 84, 24%
Elapsed time:     28m19.7s
Transferring:
 * Toy Story 4 (2019)/Toy…Atmos - KRaLiMaRKo.mkv: 17% /31.758G, 87.996M/s, 5m4Transferred:        142.535G / 651.823 GBytes, 22%, 85.367 MBytes/s, ETA 1h41m49s
Transferred:           20 / 84, 24%
Elapsed time:     28m29.7s
Transferring:
 * Toy Story 4 (2019)/Toy…Atmos - KRaLiMaRKo.mkv: 20% /31.758G, 88.764M/s, 4m5Transferred:        143.452G / 651.823 GBytes, 22%, 85.417 MBytes/s, ETA 1h41m34s
Transferred:           20 / 84, 24%
Elapsed time:     28m39.7s
Transferring:
 * Toy Story 4 (2019)/Toy…Atmos - KRaLiMaRKo.mkv: 23% /31.758G, 91.386M/s, 4m3

```
