# Task 1 & 2
This is an explanation of how each task is done, by showing what are the command used step by step 

## Task 1
This Task will focuse on Linux and shell. Task1 is devided into 10 main topics which are:
- LVM
- users, groups and permissions
- SSH
- Permissions
- SELinux
- bash script and processes
- Yum Repo
- Network management
- Cronjob
- Mariadb

### Part 1: LVM
This task involves configuring LVM on the second disk by creating a volume group with a 16 MB extent size and a logical volume of 50 extents. The logical volume is formatted as ext4 and set to mount automatically at `/mnt/data`.
#### Step by step:

**1. Prepare the Disk**
```bash
# Show all the partitions, disks and LVM volumes in the system 
lsblk
```


**2. Create a partition**
```bash
# fdisk: create, delete, or modify disk partitions
# n: create new partition
# t: change partition type to LVM
# last sector: +3G (3 GiB size)
fdisk /dev/sdb
```


**3. Create Physical Volume (PV)**
```bash
# pvcreate initializes a disk or partition to be used as a physical volume (PV) in LVM.
pvcreate /dev/sdb1
```


**4. Create Volume group (VG)**
```bash
# Creates a new LVM volume group named vg_data using the physical volume /dev/sdb1.
# -s 16M sets the physical extent size of the volume group to 16 MB
vgcreate -s 16M vg_data /dev/sdb1
```


**5. Create Logical Volume (LV)**
```bash
# Creates a logical volume named lv_data inside a volume group.
# -n lv_data sets the logical volume name to lv_data
# -l 50 allocates 50 logical extents
# vg_data the volume group nam
lvcreate -n lv_data -l 50 vg_data
```


**6. Create Ext4 Filesystem**
```bash
# formats the logical volume lv_data with the ext4 filesystem.
mkfs.ext4 /dev/vg_data/lv_data
```


**7. Mount the LV**
```bash
# Mount the filesystem to /mnt/data
mount /dev/vg_data/lv_data /mnt/data 
```

**8. Configure Automatic Mount**
```bash
# Add the LV to /etc/fstab for automatic mounting at boot
/dev/vg_data/lv_data      /mnt/data     ext4     defaults    0 0
```
```bash
# Unmount any previous mounts and mount all filesystems in fstab
umount /mnt/data
mount -a
```






---




### Part 2: Users, Groups and Permissions 
This task involves creating users with specific settings:

- `user1`  
  - Non-interactive shell (no SSH access)  
  - Added to `TrainingGroup`  
  - Password: `redhat`

- `user2` and `user3`  
  - Additional group: `admin`  
  - Password: `redhat`  
  - `user3` has root permissions
 
#### Step by step:
 
**1. Create user1**
```bash
# Create user1 with UID 601 and a non-interactive shell
useradd -u 601 -s /sbin/nologin user1 2> /dev/null

# Set the password of user1 to "redhat"
echo "redhat" | passwd -s user1 2> /dev/null

# Verify that user1 was created by checking /etc/passwd
more /etc/passwd | grep user1
```

**2. Create TrainingGroup and add user1 into it**
```bash
# Create a new group named 'TrainingGroup'
groupadd TrainingGroup

# Add user1 to the 'TrainingGroup'
useradd -aG TrainingGroup user1

# Verify that 'TrainingGroup' exists and check its members
more /etc/group | grep TrainingGroup
```


**3. Create admin group**
```bash
# Create a new group named 'admin'
groupadd admin

#Verify that 'admin' group exists
more /etc/admin | grep admin
```

**4.  Create user2&3 and assign them in admin group**
```bash
# Create user 2&3 and add them to the 'admin' group
useradd -G admin user2
useradd -G admin user3

# Set the password of user 2&3 to "redhat"
echo "redhat" | passwd -s user2
echo "redhat" | passwd -s user3

# Verify that user 2&3 are added to 'admin' group
more /etc/admin | grep admin
```

**5. Give user3 root permissions**
```bash
# Add user3 to the 'Wheel' group
usermod -aG Wheel user3

# Verify that user3 is now a member of the Wheel group
more /etc/group | grep Wheel
```





---



### Part 3: SSH 
This task involves setting up passwordless SSH access by generating an SSH key pair on one VM and copying the public key to another VM, allowing secure login without entering a password.

#### Step by step:

**1. Generate SSH Key on Local VM**
```bash
# Generate an SSH key pair using the RSA algorithm
ssh-keygen -t rsa
```


**2.  Copy Public Key to Remote VM**
```bash
# Switch to user3 to perform SSH setup
su - user3

# Copy user3's public SSH key to the remote VM (root@ipaserver.example.com)
ssh-copy-id root@ipaserver.example.com
```


**3. Test Password less SSH**
```bash
# SSH to the root user on the remote VM (ipaserver.example.com)
ssh root@ipaserver.example.com
```
This test verifies that passwordless SSH works.
Even if root login is restricted by default, ensure that the SSH configuration on the remote VM under this path `/etc/ssh/sshd_config` has:

```nginx
PermitRootLogin prohibit-password
```
so that root can log in using SSH keys without a password (using keys).






---




### Part 4: Permissions

This task involves setting file permissions for specific users:

- Copy `/etc/fstab` to `/var/tmp/admin`.  
- Configure permissions so that:
  - `user1` can **read, write, and modify** the file.  
  - `user2` has **no permissions**. 


#### Step by step:

**1. Copy `/etc/fstab` to `/var/tmp/admin`**
```bash
cp /etc/fstab /var/tmp/admin
```

**2. Set ACL on each user**
```bash
# Give user1 read, write, and execute permissions on /var/tmp/admin
setfacl -m u:user1:rwx /var/tmp/admin

# Remove all permissions for user2 on /var/tmp/admin
setfacl -m u:user2:--- /var/tmp/admin
```


**3. Verify that the permission are sat correcly**
```bash
# Display the ACL of /var/tmp/admin to verify user permissions
getfacl /var/tmp/admin
```






---



### Part 5: SELinux 

**1. Open /etc/selinux/config file and change the mode into enforcing**
```bash
# Change the SELINUX variable into enforcing
SELINUX=enforcing
```

**2. Reboot and check the SELinux mode**
```bash
# Reboot the system 
systemctl reboot
# Get the system mode 
getenforce 
```





---







### Part 6: bash script and processes
This script starts a background `sleep` process for 500 seconds, tracks its PID, and monitors it for 5 seconds.  
During this time, if the user presses `k`, the script attempts to kill the background process.  
The script prints messages showing whether the process is still running, killed, or already terminated.


**1. Script for checkig if a proccess is still running for speccefic time and kill it according to the user**
```bash


#!/bin/bash

sleep 500 &
sleep_PID=$!

after_n_seconds=5;

echo "The command has started now and its PID: $sleep_PID"

current_time=$(date +%s)
end_time=$((current_time + after_n_seconds))

while [ $current_time -le $end_time ]; do
       echo "The script is still running"


       read -t 1 key
       if [[ $key == "k" ]]; then
               echo "Killing background process $sleep_PID"
              if kill "$sleep_PID" 2> /dev/null ; then
                       echo "Process killed"

               else
                       echo "Process is already killed"
              fi
       fi

       sleep 1

       current_time=$(date +%s)
done


echo "Whow killed the sleep process"

if ps -p "$sleep_PID" > /dev/null 2>&1 ; then
        kill "$sleep_PID" 2> /dev/null
        echo "The script killed it"
else
        echo "The user killed it"
fi


echo "The script finished after $after_n_seconds seconds"


```





---




### Part 7: Yum Repo

This task involves installing essential packages and setting up repositories on RHEL 10. 
It includes installing `tmux` and Apache (`httpd`), adding the MySQL repository to install the MySQL server, 
and adding the Zabbix repository to download, prepare, and install all Zabbix packages with their dependencies.


**1. Install tmux**
```bash
dnf install tmux 
```



**2. Install Apache server**
```bash
dnf install httpd 
```



**3. Install MySQL**

First, we should **install `MYSQL repo** from the internet, since MYSQL packages are not provided in the local repos (AppStream & BaseOS).
The link below is taken from the offical the webisite (https://dev.mysql.com/downloads/file/?id=542359).

```bash
dnf install https://dev.mysql.com/get/mysql84-community-release-el10-2.noarch.rpm
```
Then search of MySQL server package ,and check from which repo belong.

```bash
# Search for available MySQL packages
dnf search mysql
#  Get detailed info about the MySQL community server package and its repo
dnf info mysql-community-server
```

Finally, Install MySQL server.

```bash
dnf install mysql-community-server
```




**4. Install Zabbix packages**

**Step 1: Add the Zabbix repository**

First, **Install the `Zabbix repo`** from the internet to download (mirror) all the rpm files from it later.
And to install it we should get the repo link from the Zabbix official website provided in task 1 file, but I we habe to make sure change the version of the RHEL to make it compateble with our machine

This command is provided from the official webiste, so I did not use the usual one which is `dnf install <repo_link>`

```bash
# Download and install the Zabbix repository RPM for RHEL 10
# This adds the Zabbix 7.0 repository to the system so we can install Zabbix packages via DNF
rpm -Uvh https://repo.zabbix.com/zabbix/7.0/rhel/10/x86_64/zabbix-release-latest-7.0.el10.noarch.rpm
```


**Step 2: Sync repository packages locally**


Then, `Download all the rpm files` from the repo on the internet

```bash
reposync -p /zabbix --repoid=zabbix --download-metadata
```


**Step 3: Identify dependencies**

After that, we need to download all required dependencies, since some packages are not available in the local repository.  
First, we identify the required package names by attempting to install the needed packages (such as `zabbix-server-sql`, `zabbix-agent`, etc.).  
During the installation process, the missing dependencies are displayed along with the repositories they belong to.


```bash
dnf install zabbix-server zabbix-server-mysql zabbix-sql-scripts zabbix-agent zabbix-web-mysql --enablerepo="*"
```

download the dependcies.
```bash
dnf --enablerepo=appstream --enablerepo=baseos download nginx-filesystem php-bcmath php-common php-fpm php-gd php-ldap php-mbstring php-mysqlnd php-xml OpenIPMI-libs unixODB
```


**Step 4: Prepare the local repository**
 
The installation process cannot be completed without knowing which repositories provide the required packages and their dependencies.  
Therefore, we must create or update the repository metadata to ensure all necessary packages and dependencies are available.

```bash
# Create or update repository metadata for the local Zabbix repository directory
createrepo /zabbix/
```



**Step 5: Install Zabbix packages from local repo**


Finally, install all the desired packages.
```bash
dnf install zabbix-server zabbix-server-mysql zabbix-sql-scripts zabbix-agent zabbix-web-mysql --disablerepo="*" --enablerepo=zabbix 
```








--- 







### Part 8: Network management

This task focuses on basic network management by configuring the firewall. 
It includes opening ports 80 and 443, making these changes permanent, 
and blocking SSH access from a specific colleague's IP address to the VM.


**1. Open Port 80 & 443 and make the changes permanent**
```bash
# Open port 80 and 443 permanently
firewall-cmd --permanent --add-port=80/tcp
firewall-cmd --permanent --add-port=443/tcp

# Apply the changes
firewall-cmd --reload

# Verify all open ports and settings
firewall-cmd --list-all
```



**2. Add rich rule to prevent specific IP**
```bash
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.99.13" service name="ssh" reject'
```






---







### Part 9: Cronjob

This task involves creating a cronjob that runs daily at 1:30 AM to record all currently logged-in users. 
The output is saved to a file in the format `timestamp – users`. 
The cronjob can execute a shell script to automate this task.



**1. Edit the crontab for the root user**

To schedule the job at a specific time, we use `crontab`. 
Add this line `30 1 * * * /task_script.sh`
```bash
# -e: opens the crontab editor to add or modify jobs.
crontab -e

# -l: lists existing cron jobs.  
crontab -l
```

**2. Shell script to print the logged users with into a log file**

This script collects a list of all currently logged-in users and appends it to a log file (`/var/log/user_logs.log`) along with the current timestamp. 
It ensures the usernames are unique, separated by commas, and if no users are logged in, it records "No users logged in". 
Each execution adds a new entry in the format: `timestamp – users`.


```bash
output_file=/var/log/user_logs.log
time=$(date)

users=$(who | cut -d ' ' -f 1 | sort -u | tr '\n' ',' | sed 's/,$//')

if [ -z "$users" ]; then
        users="No users logged in"
fi

echo "$time - $users" >> "$output_file"
```








---









### Part 10: MariaDB

This task involves setting up MariaDB on the system using a local repository. 
It includes opening the necessary ports in the firewall, creating a database and user with proper permissions, 
and connecting to the database using the new user. 
As an example, a `mahmouddb` database is created with ten student records containing personal details, program enrolled, expected graduation year, and student numbers.


**1. Install MariaDB from the local repo that was created earlier**

Befor installing the MariaDB from the local repo, we have to make sure that all the dependencies are all exist in the repo, so first we should download all the dependencies packages in the local ropo under this path `/zabbix`


**Download the dependencies packages**

```bash
# downloads the 'mariadb' and 'mariadb-server' packages along with all their dependencies without installing them.
dnf download --resolve mariadb-server mariadb
```

**Install MariaDB**

```bash
dnf install mariadb-server --disablerepo="*" --enablerepo="zabbix"
```




**2. Open ports in the iptables from MariaDB**

I used `firewall` to add a port since there is no `iptables` in the Redhat course

```bash
# Opne the default port of MariaDB which is 3306 over TCP
firewall-cmd --permanent --add-port=3306/tcp4
firewall-cmd --reload
```



**3. Create database, user (note: handle permissions)**

First, log in as the MariaDB root user. The root user can manage databases, create tables, and add new users:


```bash
# Log in as root user
# -u specifies the username
mysql -u root
```

After logging in, you can execute the following SQL queries to manage the database and user:
```bash
# Create a database named mahmoudDB
CREATE DATABASE mahmoudDB;

# Create a user named 'user' with password 'password', accessible only from localhostwith password: password
CREATE USER 'user'@'localhost' IDENTIFIED BY 'password';

# Grant all privileges on the database to the new user
GRANT ALL PRIVILEGES ON mahmoudDB.* TO 'user'@'localhost';

# Apply the changes
FLUSH PRIVILEGES;

```


**4. Connect to the database created in step 3 using the new user (with password)**
```bash
# login as 'user' user and access the mahmoudDB database with password
mysql -u user -p mahmoudDB
```




**4. Create a schema**

To create a schema, we define a table with a meaningful name (for example, `students`). 
Each column must have an appropriate data type, and a suitable column should be selected as the primary key to uniquely identify each record.

```bash
CREATE TABLE students (
    student_number VARCHAR(10) PRIMARY KEY,
    firstname VARCHAR(50),
    lastname VARCHAR(50),
    program VARCHAR(50),
    grad_year INT
);

```

This SQL statement inserts sample student records into the `students` table.  
Each row represents one student with a unique student number, first and last name, enrolled program, and expected graduation year.

```bash
INSERT INTO students (student_number, firstname, lastname, program, grad_year) VALUES
('110-001', 'Allen',  'Brown',  'mechanical',      2017),
('110-002', 'David',  'Brown',  'mechanical',      2017),
('110-003', 'Mary',   'Green',  'mechanical',      2018),
('110-004', 'Dennis', 'Green',  'electrical',      2018),
('110-005', 'Joseph', 'Black',  'electrical',      2018),
('110-006', 'Dennis', 'Black',  'electrical',      2020),
('110-007', 'Ritchie','Salt',   'computer science',2020),
('110-008', 'Robert', 'Salt',   'computer science',2020),
('110-009', 'David',  'Suzuki', 'computer science',2020),
('110-010', 'Mary',   'Chen',   'computer science',2020);
```





**4.  Verify that the schema is created successfully**
```bash
# Display all records from the students table
SELECT *
FROM students;
```
















---





## Task 2

This task focuses on monitoring system performance by automatically collecting `CPU`, `memory`, and `disk` statistics, processing them to calculate averages, and displaying the results on a web page using an Apache server.


### Part 1: Data Collection Shell Script

This Bash script is responsible for collecting system performance statistics and storing them for later analysis and web display. It creates a directory inside the Apache web root to hold the data files and generates a timestamp to uniquely identify each collection cycle.

The script collects:
- **CPU statistics** by extracting the CPU idle percentage.
- **Memory statistics** including used and free memory in gigabytes.
- **Disk statistics** including used and free disk space for the root (`/`) partition.

Each metric is stored in a separate file named with the metric type and the timestamp.

```bash
#!/bin/bash

################################## collect data ######################################
output_dir="/var/www/html/data"
mkdir -p $output_dir

time=$(date +%Y-%m-%d-%H-%M-%S)

cpu=$(top -bn1 | grep "Cpu" | grep -o '[0-9.]* id' | cut -d' ' -f1)
echo "$time $cpu" >> "$output_dir/cpu_$time.txt"

used_mem=$(free --giga | grep "Mem" | tr -s ' ' | cut -d' ' -f3)
free_mem=$(free --giga | grep "Mem" | tr -s ' ' | cut -d' ' -f7)
echo "$time $used_mem $free_mem" >> "$output_dir/mem_$time.txt"

used_disk=$(df -BG / | tail -1 | tr -s ' ' | cut -d' ' -f3 | sed 's/G//')
free_disk=$(df -BG / | tail -1 | tr -s ' ' | cut -d' ' -f4 | sed 's/G//')

echo "$time $used_disk $free_disk" >> "$output_dir/disk_$time.txt"

echo "All the data r collected at time: $time"

```


### Part 2: First Cron Job: Periodic Data Collection

This cronjob will work every minute to collect the data from the CPU, memory and disk.
To make the cronjob to collect data every minute add this record `* * * * *  `/collect.sh`

```bash
# To add a new record in the cron tab
# Add this record `* * * * *  /collect.sh`
crontab -e

# To ensure that the record was added successfully
crontab -l
```





---


### Part 3: Data Processing and Average Calculation Shell Script

This script processes the system statistics collected by the data collection script. It reads CPU, memory, and disk data from stored files, calculates average values, and dynamically generates HTML pages to display both the averages and all recorded data with timestamps.



---


#### 3.1 Reading Collected Data from Files

The script reads previously collected system statistics from text files stored in the `/var/www/html/data` directory.  
Each metric (CPU, memory, and disk) is stored in separate files with a timestamp, allowing the script to iterate over all available records for processing.













