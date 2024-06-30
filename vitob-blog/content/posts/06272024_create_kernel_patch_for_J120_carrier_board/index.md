+++
title = 'Create Kernel Patch for TX2 J120 Carrier Board'
date = 2024-06-29T21:05:00-07:00
draft = false
toc = true
+++


## Patch Linux Kernel using Official Firmware

J120 is a third-party Jetson TX2 carrier board from Auvidea, thus whenever new Jetpack was released there need updated firmware from Auvidea. This section describes how to apply the J120 updated [firmware v3.0](https://auvidea.eu/firmware/) released on Feb 2020 (download [link](https://auvidea.eu/download/firmware/J120/J90-J120-J130_4_3.tar.bz2)):

```
Feb 2020 	J90/J120/J130 (202 MB) 	3.0 

supports: Jetson TX2 only
– JetPack 4.3 (L4T 32.3.1)
– 2x USB 3.0
– IMU MPU9250 (spidev1.0)
– 1x GbE
– 1x M.2 NVME PCIe x4
– port mapping: config 2 (default)
```

Unzip the downloaded file shows two tar'ed packages: kernel_out and kernel_src. The former is already-built binaries in the relase; the latter contains the corresponding modified linux kernel source files that the former binaries were generated. 

The steps to apply the patches are listed in file How_to_flash_TX2_with_nvidia_sdkmanager.txt as below. It uses the released patched binaries only. 

```
How to install the auvidea kernel the nvidia sdkmanager
####################################################################################
This manual is written for the Jetson TX2


1. prepare the sdk
ONLY NEEDED IF NO TX2 WAS FLASHED BEFORE
-start the sdkmanager
-select the Jetson TX2(P3310) as target Hardware
-select Jetpack 4.3 as target operating system
-start the installation until the sdk asks you for either use automatic or manual setup
 -> at this point you can choose to "skip" the rest of the installation and continue with step 2 of this instructions

2. copy the contents of the kernel_out folder in the auvidea packet to the nvida_sdk folder
-> cp ~/kernel_out/* /home/USER/nvidia/nvidia_sdk/JetPack_4.3_Linux_P3310/
 
3. switch in the TX2 folder and apply binaries 
-> cd /home/USER/nvidia/nvidia_sdk/JetPack_4.3_Linux_P3310/Linux_for_Tegra/
-> sudo ./apply_binaries.sh

4. start the sdkmanger and follow the normal installation process



You can also use the flowing commands to flash the TX2 after Step 3:

cd /home/USER/nvidia/nvidia_sdk/JetPack_4.3_Linux_P3310/Linux_for_Tegra/

```


## Verify modified kernel source files

### Download correct version 

```
./source_sync.sh -k tegra-l4t-r32.3.1
```

### Create a special patch file

Let's call original unchanged kernel source directory `target_directory`, the directory contains patched sources `source_directory`. 

The condition is a bit complex: 
1) There exist files in `target_directory` that don't need to be modified and these files don't exist in `source directory`;
2) There might exist new files/directories in `source directory` that don't exist in `target directory`

A simple `diff` to generate patch file is not possible. 

A script `generate_patch.sh` is created for the purpose:
```
#!/bin/bash

# Check if both arguments are provided
if [ "$#" -ne 2 ]; then
    echo "Usage: $0 <source_directory> <target_directory>"
    exit 1
fi

SOURCE_DIR="$1"
TARGET_DIR="$2"

# Function to generate diff for a file
generate_file_diff() {
    local file="$1"
    local rel_path="${file#$SOURCE_DIR/}"
    if [ -f "$TARGET_DIR/$rel_path" ]; then
        diff -uN "$TARGET_DIR/$rel_path" "$file"
    else
        diff -uN /dev/null "$file"
    fi
}

# Function to process a directory
process_directory() {
    local dir="$1"
    local rel_dir="${dir#$SOURCE_DIR/}"

    # Create directory entry if it doesn't exist in target
    if [ ! -d "$TARGET_DIR/$rel_dir" ] && [ "$rel_dir" != "" ]; then
        echo "diff -uN $TARGET_DIR/$rel_dir $SOURCE_DIR/$rel_dir"
        echo "--- $TARGET_DIR/$rel_dir"
        echo "+++ $SOURCE_DIR/$rel_dir"
        echo "@@ -0,0 +1 @@"
        echo "+$rel_dir/"
    fi

    # Process files in this directory
    find "$dir" -maxdepth 1 -type f | while read -r file; do
        generate_file_diff "$file"
    done

    # Recursively process subdirectories
    find "$dir" -mindepth 1 -maxdepth 1 -type d | while read -r subdir; do
        process_directory "$subdir"
    done
}

# Start processing from the root of the source directory
process_directory "$SOURCE_DIR" > update.patch
```

Run this script to generate `update.patch` file:
```
xxx@ubuntu18:~/tx2-j120/J90-J120-J130_4_3/kernel_src$ pwd
/home/xxx/tx2-j120/J90-J120-J130_4_3/kernel_src    <= This dir is from Auvidea firmware v3.0
xxx@ubuntu18:~/tx2-j120/J90-J120-J130_4_3/kernel_src$ bash ../generate_patch.sh . ~/tx2_source_build/r32-3-1_Release_v1.0/sources
```

### Apply the patch file

We need another special script `apply_patch.sh` because we need first create those non-exist new directories in `target directory` first:
```
#!/bin/bash

PATCH_FILE="update.patch"

# Check if both arguments are provided
if [ "$#" -ne 1 ]; then
    echo "Usage: $0 <target_directory>"
    exit 1
fi

TARGET_DIR="$1"



# Extract directory creations
grep -E "^\+[^+]\S+\w\/$" "$PATCH_FILE" | sed 's/^+//' | grep "/$" > directories.txt

# Create directories
while read -r dir; do
    mkdir -p "$TARGET_DIR/$dir"
done < directories.txt

# Remove directory creation diffs from the patch file
sed -i '/^diff.*\/$/d; /^---.*\/$/d; /^+++.*\/$/d; /^@@ -0,0 +1 @@$/d; /^\+[^+]\S+\w\/$/d' "$PATCH_FILE"

# Apply the modified patch
patch -p1 -N -d "$TARGET_DIR" < "$PATCH_FILE"

```

Then apply the patch:
```
bash ../apply_patch.sh ~/tx2_source_build/r32-3-1_Release_v1.0/sources
```

### Compile the kernel and verify

Skip here the manual process. 


##  Create patch from modified kernel source files

After we've verified the source file changes, we can leverage Yocto `devtool` to generate the patch file. How to get the Linux kernel sources and use `devtool` can be found in previous blogs. Skipping here. The key part is to get the source files patched using the two special scripts (`generate_patch.sh` and `apply_patch.sh` in last section of this blog article. 

The patch is recorded in [this commit](https://github.com/jinchenglee/meta-tx2-jp43/commit/e30ec5e1c176861238add59cd4ead200e6f99043) into branch `dev/tx2-j120-jp43` of [meta-tx2-jp43](https://github.com/jinchenglee/meta-tx2-jp43). The key patch is [0003-Linux-kernel-patch-for-TX2-on-J120-carrier-board-wit.patch](https://github.com/jinchenglee/meta-tx2-jp43/blob/dev/tx2-j120-jp43/recipes-kernel/linux/linux-tegra/0003-Linux-kernel-patch-for-TX2-on-J120-carrier-board-wit.patch). 
