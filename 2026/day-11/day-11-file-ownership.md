# File Ownership (chown & chgrp)

## Understanding Ownership
- Run `ls -l` in your home directory

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/2fece9d7db0b1345b20b6c86f26b60d3f7681b00/2026/day-11/Images/Screenshot%20(487).png)

    `-rw-r--r-- 1 owner group size date filename`

    -   Owner → Individual user with direct control.

    -   Group → Multiple users who may share access.

## Files & Directories Created
- Basic `chown` Operations

    `touch devops-file.txt`

    `ls -l devops-file.txt`

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/2fece9d7db0b1345b20b6c86f26b60d3f7681b00/2026/day-11/Images/Screenshot%20(488).png)

    `sudo chown tokyo devops-file.txt`

    `ls -l devops-file.txt`

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/2fece9d7db0b1345b20b6c86f26b60d3f7681b00/2026/day-11/Images/Screenshot%20(489).png)

    `sudo chown berlin devops-file.txt`

    `ls -l devops-file.txt`

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/2fece9d7db0b1345b20b6c86f26b60d3f7681b00/2026/day-11/Images/Screenshot%20(490).png)

- Basic `chgrp` Operations

    `touch team-notes.txt`

    `ls -l team-notes.txt`

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/2fece9d7db0b1345b20b6c86f26b60d3f7681b00/2026/day-11/Images/Screenshot%20(491).png)

    `sudo groupadd heist-team`

    `sudo chgrp heist-team team-notes.txt`

    `ls -l team-notes.txt`

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/2fece9d7db0b1345b20b6c86f26b60d3f7681b00/2026/day-11/Images/Screenshot%20(492).png)

## Ownership Changes
- Combined Owner & Group Change

    `touch project-config.yaml`

    `sudo chown professor:heist-team project-config.yaml`

    `mkdir app-logs`

    `sudo chown berlin:heist-team app-logs`

    `ls -l`

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/2fece9d7db0b1345b20b6c86f26b60d3f7681b00/2026/day-11/Images/Screenshot%20(494).png)

- Recursive Ownership
    
    `mkdir -p heist-project/vault`
    
    `mkdir -p heist-project/plans`

    `touch heist-project/vault/gold.txt`

    `touch heist-project/plans/strategy.conf`

    `sudo groupadd planners`

    `sudo chown -R professor:planners heist-project/`

    `ls -lR heist-project/`

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/2fece9d7db0b1345b20b6c86f26b60d3f7681b00/2026/day-11/Images/Screenshot%20(495).png)

- Practice Challenge

    -   Create users
        
        `sudo useradd tokyo`

        `sudo useradd berlin`

        `sudo useradd nairobi`

    -   Create groups
    
        `sudo groupadd vault-team`

        `sudo groupadd tech-team`

    -   Create directory

        `mkdir bank-heist`
    
    - Create 3 files inside

    `touch bank-heist/access-codes.txt`

    `touch bank-heist/blueprints.pdf`

    `touch bank-heist/escape-plan.txt`

    - Set different ownership

    `sudo chown tokyo:vault-team bank-heist/access-codes.txt`

    `sudo chown berlin:tech-team bank-heist/blueprints.pdf`

    `sudo chown nairobi:vault-team bank-heist/escape-plan.txt`

    -   Verify

    `ls -l bank-heist/`

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/2fece9d7db0b1345b20b6c86f26b60d3f7681b00/2026/day-11/Images/Screenshot%20(496).png)

## Commands Used
1. View ownership

    `ls -l filename`

2. Change owner only

    `sudo chown newowner filename`

3. Change group only

    `sudo chgrp newgroup filename`

4. Change both owner and group

    `sudo chown owner:group filename`

5. Recursive change (directories)

    `sudo chown -R owner:group directory/`

6. Change only group with chown

    `sudo chown :groupname filename`


## What I Learned
1. Dual Ownership Model  
    Every file in Linux has two ownership attributes:
    - Owner (user): The individual account that created or owns the file.
    - Group: A collection of users who may share access to the file. This ensures collaborative work without giving full control to everyone. 

2. Ownership Controls Access & Security  
    File ownership, combined with permissions, determines who can read, write, or execute a file. This is a cornerstone of Linux’s multi-user security model, preventing unauthorized access or modification. 

3. Ownership Can Be Changed with Commands  
    Administrators can reassign ownership using:
    - chown → change file owner (and optionally group).
    - chgrp → change group ownership.
    - Recursive flags (-R) allow applying changes across entire directories. These commands are essential for managing shared projects and system security.