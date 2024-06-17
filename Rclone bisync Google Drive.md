# Tutorial: Sync Google Drive files on Linux with Rclone bisync and systemd 
If you are using Google Drive (or similar cloud storage) for your decoy activities when trying to appear as a normal person, then this tutorial may be useful for you. 

Google Drive doesn't provide native bi-directional file syncing application on Linux and almost all solutions either suck or are paid. One free option is Rclone - Rclone is a great project, but the configuration interface is utterly horrible (if there's interest I can write down all the things that are wrong with it), so in this tutorial I'll try to provide a specific pre-configured setup to make this hopefully less painful. As you can see the following steps are mostly manual and runnable by a copypaste woodpecker, so this will hopefully automated away in the future.

**High level summary**
- Install rclone 
- Create Google Drive "remote" connection (called "drive")
- Create filter file to ignore some files from syncing
- Create a command using new feature called `rclone bisync` and run this command for the first time
- [optional] Create systemd service to run the same command for you
- [optional] Create systemd timer to run the service regularly
- [optional] Create systemd path watcher to run the service automatically when you change the files

**Files & Folders**
- `$HOME/.config/rclone/rclone.conf` - rclone config for your connection 
- `$HOME/.config/systemd/user/` - folder to put your systemd services and watchers (and rclone filters file for our case)
- `$HOME/sync/` - folder where we put the actual files. 

**Common commands & Required knowledge**
- Create and edit a file `nano ~/.config/SOMEFILE` (save with `Ctrl+O` and close with `Ctrl+X`)

## Rclone Installation
To use the new `rclone bisync` feature you need to get a new version v1.66+ of rclone. This version is not currently available in your usual repositories/packaging systems, so you need to install it manually. 
If you are coming from the future, then try running e.g. `sudo apt install rclone`.

Go to https://rclone.org/downloads/ and download the appropriate Release binary (most likely "Intel/AMD - 64 Bit" and ".deb") to any folder on your disk. Try double clicking on the downloaded .deb file and that may open installer depending on your Linux distribution. If not, then open terminal in the same folder and run:
```sh
sudo dpkg -i rclone-*.deb
```
(this may again vary based on your Linux distribution and packaging system)

## Create Google Drive Connection
Unfortunatelly for this you will have to follow another tutorial.
**NAME YOUR CONNECTION "drive" or update all further commands accordingly**

These are decent:
- https://ostechnix.com/mount-google-drive-using-rclone-in-linux/
- https://youtu.be/YDF1nBaAptw?t=291
- or the rclone documentation: https://rclone.org/drive/

Summary
- run `rclone config` and follow the steps to add new remote for Google Drive
- Name your remote "drive"
- there are many footguns in that process. I'm sorry.

## Create the filters file
When syncing files it's useful to filter out some files that you don't want to be synced. For example hidden files that start with ".". Run the following command to create this file. The documentation is quite bad, but you can learn how to set up filtering here: https://rclone.org/filtering/.

Create the config folder for your systemd programs:
```sh
mkdir -p ~/.config/systemd/user/
```

Create `~/.config/systemd/user/rclone.filter.txt` and paste:
```sh
# Filter file for use with bisync
# See https://rclone.org/filtering/ for filtering rules
# NOTICE: If you make changes to this file you MUST do a --resync run.
#         Run with --dry-run to see what changes will be made.

# Exclude all hidden files and folders
- .* 
- .*/**

# Exclude temp files
- ~*.tmp
```

## First time running rclone bisync command
Now we can finally run the sync.

We need to do this in 3 phases:
- First run with `--resync --dry-run` params to see outputs, but not change anything on your system so you can verify it works correctly.
- Second run with `--resync`  which triggers the initial redownload of the data from Google Drive
- Third run without these params to check that the "normal operation" works.

Steps
1. Create the folder on your Linux where you want to sync files.
	- `mkdir -p ~/sync`
	- (also make sure you have the systemd config folder `mkdir -p ~/.config/systemd/user/`)
2. First time run the command with --resync
```sh
/usr/bin/rclone bisync drive: ~/sync
    -MvP \
    --create-empty-src-dirs \
    --compare size,modtime,checksum \
    --filters-file ~/.config/systemd/user/rclone.filter.txt \
    --conflict-resolve newer \
    --conflict-loser delete \
    --conflict-suffix sync-conflict-{DateOnly}- \
    --suffix-keep-extension \
    --resilient \
    --recover \
    --no-slow-hash \
    --drive-skip-gdocs \
    --fix-case \
    --resync \
    --dry-run
```
2. Now run the same command, but remove the last `--dry-run` param (and remove the backslash on the previous line). 
	- Check that your files have been correctly downloaded to your selected folder. 
3. Now run the same command, but remove both `--dry-run` and `--resync` (and the two backslashes).
	- Check that everything looks as intended, both on your local file system and in your Google Drive. 

## Create the service
Create `~/.config/systemd/user/rclone.service` and paste:

```sh
[Unit]
Description=Syncing local files with Google Drive using rclone bisync
Documentation=man:rclone(1)
After=network-online.target
Wants=network-online.target 
StartLimitIntervalSec=60
StartLimitBurst=1

[Service]
Type=oneshot
ExecStart= \
/usr/bin/rclone bisync drive: %h/sync
    -MvP \
    --create-empty-src-dirs \
    --compare size,modtime,checksum \
    --filters-file %h/.config/systemd/user/rclone.filter.txt \
    --conflict-resolve newer \
    --conflict-loser delete \
    --conflict-suffix sync-conflict-{DateOnly}- \
    --suffix-keep-extension \
    --resilient \
    --recover \
    --no-slow-hash \
    --drive-skip-gdocs \
    --fix-case
	
[Install]
WantedBy=default.target
```

Note: In the service we ensure that it runs when we are online and that it only starts once in 60 seconds (this is useful when used with file watcher later).
#### ENABLE AND RUN THE SERVICE
```shell
# Read and reload all unit configs. Required so it gets new config file state.
systemctl --user daemon-reload 
# Enable to run after reboot and also start immediately with "--now"
```
#### HELPFUL COMMANDS
```sh
# See status
systemctl --user status rclone
# See the service logs
journalctl --user -xeu rclone
# Stop the service (but won't disable after next reboot)
systemctl --user stop rclone
# Disable the service
systemctl --user disable rclone
```

## Create the timer
Following [tutorial](https://medium.com/@rockibul.islam20/configure-and-implement-of-systemd-timers-8c640cc6d667)
Create `~/.config/systemd/user/rclone.timer` with:
```sh
[Unit]  
Description=Rclone timer (triggers every 10 minutes)
  
[Timer]  
OnCalendar=*:0/10
Persistent=true  
Unit=rclone.service  
  
[Install]  
WantedBy=default.target
```

#### ENABLE AND RUN THE TIMER
```sh
systemctl --user daemon-reload
systemctl --user enable --now rclone.timer
```
#### HELPFUL COMMANDS
```sh
# See timer logs
journalctl --user -xeu rclone.timer
# List all timers
systemctl --user list-timers --all 
# See the log for the timer
journalctl --user -xeu rclone.timer
# Stop the timer
systemctl --user stop rclone.timer
# Disable the timer
systemctl --user disable rclone.timer
```


## Create the file watcher
Following [stackoverflow](https://superuser.com/questions/1171751/restart-systemd-service-automatically-whenever-a-directory-changes-any-file-ins). This is useful to automatically trigger the sync as soon as the files on your disk change. 
Create `~/.config/systemd/user/rclone.path` with:
```sh
[Path]
Unit=rclone.service
PathChanged=%h/sync
# trigger on changes to a file not just create/delete

[Install]
WantedBy=default.target
```

#### ENABLE AND RUN THE FILE WATCHER
```sh
systemctl --user daemon-reload 
systemctl --user enable --now rclone.{timer,path,service}
```

#### SEE THE STATUS OF THE COMPLETE SETUP
```sh
systemctl --user status rclone.service rclone.timer rclone.path
# Combined logs. 
journalctl --user -xeu rclone.service rclone.timer rclone.path
```