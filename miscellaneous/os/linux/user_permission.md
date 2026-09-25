# Linux System 

## User and Group Management 

## User Management

### Creating a User
- **Create a new user**:
  ```bash
  sudo useradd username
  ```
  Example: Creates a user with the name `username`.

- **Create a user with a home directory**:
  ```bash
  sudo useradd -m username
  ```
  Example: Creates a user and generates a home directory (`/home/username`).

- **Assign a password to a user**:
  ```bash
  sudo passwd username
  ```
  Example: Sets a password for the user `username`.

### Modifying a User
- **Modify an existing user**:
  ```bash
  sudo usermod -l new_name old_name
  ```
  Example: Changes the username from `old_name` to `new_name`.

- **Add a user to a group**:
  ```bash
  sudo usermod -aG group_name username
  ```
  Example: Adds `username` to the group `group_name`.

### Deleting a User
- **Delete a user**:
  ```bash
  sudo userdel username
  ```
  Example: Deletes the user `username`.

- **Delete a user and their home directory**:
  ```bash
  sudo userdel -r username
  ```
  Example: Deletes the user and their home directory (`/home/username`).

## Group Management

### Creating and Managing Groups
- **Create a group**:
  ```bash
  sudo groupadd group_name
  ```
  Example: Creates a group named `group_name`.

- **Add a user to a group**:
  ```bash
  sudo usermod -aG group_name username
  ```
  Example: Adds `username` to the group `group_name`.

- **Change a user's primary group**:
  ```bash
  sudo usermod -g group_name username
  ```
  Example: Sets `group_name` as the primary group for `username`.

### Deleting a Group
- **Delete a group**:
  ```bash
  sudo groupdel group_name
  ```
  Example: Deletes the group `group_name`.


## Permissions and Ownership Management

### Change Ownership of a File or Directory
- **Change the owner of a file or directory**:
  ```bash
  sudo chown username file_or_directory
  ```
  Example: Changes the owner of `file_or_directory` to `username`.

- **Change the owner and group**:
  ```bash
  sudo chown username:group_name file_or_directory
  ```
  Example: Changes the owner to `username` and the group to `group_name`.

### Change Permissions
- **Change the permissions of a file or directory**:
  ```bash
  sudo chmod 755 file_or_directory
  ```
  Example: Sets specific permissions (`rwxr-xr-x`) on `file_or_directory`.

- **Recursively change permissions on a directory**:
  ```bash
  sudo chmod -R 755 directory
  ```
  Example: Applies `755` permissions to all files and subdirectories within `directory`.

## User and Group Information

### View User Information
- **Display information about a user**:
  ```bash
  id username
  ```
  Example: Shows the user ID and group details for `username`.

- **See currently logged-in users**:
  ```bash
  who
  ```
  Example: Displays users currently logged into the system.

- **Display the last logged-in users**:
  ```bash
  last
  ```
  Example: Lists the last users who logged in.

### View Available Groups
- **List all groups**:
  ```bash
  getent group
  ```
  Example: Displays all groups available on the system.

### Modify User and Group Configuration Files
- **User file (/etc/passwd)**:
  ```bash
  sudo nano /etc/passwd
  ```
  Example: Contains user information.

- **Group file (/etc/group)**:
  ```bash
  sudo nano /etc/group
  ```
  Example: Contains group information.

## Security and Advanced Management

### Lock and Unlock a User Account
- **Lock a user account**:
  ```bash
  sudo usermod -L username
  ```
  Example: Locks the account `username`.

- **Unlock a user account**:
  ```bash
  sudo usermod -U username
  ```
  Example: Unlocks the account `username`.

### Password Expiration
- **Set password expiration**:
  ```bash
  sudo chage -E YYYY-MM-DD username
  ```
  Example: Sets the password expiration date for `username`.

- **Check password expiration settings**:
  ```bash
  sudo chage -l username
  ```
  Example: Displays expiration settings for `username`.

## Linux File Permissions 

## Basic Concepts

- **File Permissions**: Each file or directory in Linux has associated permissions that determine who can read, write, or execute the file.

- **User Types**:
  - **Owner (u)**: The user who owns the file.
  - **Group (g)**: Users who are members of the file's group.
  - **Others (o)**: All other users on the system.

- **Permission Types**:
  - **Read (r)**: Permission to read the file or list the directory contents.
  - **Write (w)**: Permission to modify the file or directory.
  - **Execute (x)**: Permission to execute the file (if it’s a script or binary) or access the directory.

## Viewing Permissions

- **View file permissions**:
  ```bash
  ls -l
  ```
  *Example:* Displays a long listing of directory contents, including permissions, owner, group, and other details.

  Output:
  ```
  -rwxr-xr-- 1 user group 1024 Aug 16 08:00 myscript.sh
  ```
  *Explanation:* The first character represents the file type (`-` for regular files, `d` for directories). The next nine characters are the permissions for owner, group, and others.

## Changing Permissions

- **Using `chmod` command**:
  - **Change permissions using symbolic mode**:
    ```bash
    chmod u+rwx,g+rx,o-r myfile.txt
    ```
    *Example:* Grants read, write, and execute permissions to the owner, read and execute permissions to the group, and removes read permission for others.

  - **Change permissions using numeric mode (octal)**:
    ```bash
    chmod 755 myscript.sh
    ```
    *Example:* Sets the permissions to `rwxr-xr-x` (owner can read, write, execute; group and others can read and execute).

  - **Make a file executable**:
    ```bash
    chmod +x myscript.sh
    ```
    *Example:* Adds execute permissions to the file for everyone.

  - **Remove write permission for group and others**:
    ```bash
    chmod go-w myfile.txt
    ```
    *Example:* Removes write permission for the group and others.

## Changing Ownership

- **Using `chown` command**:
  - **Change file owner**:
    ```bash
    sudo chown newowner myfile.txt
    ```
    *Example:* Changes the owner of `myfile.txt` to `newowner`.

  - **Change file owner and group**:
    ```bash
    sudo chown newowner:newgroup myfile.txt
    ```
    *Example:* Changes the owner to `newowner` and the group to `newgroup`.

  - **Change group ownership**:
    ```bash
    sudo chgrp newgroup myfile.txt
    ```
    *Example:* Changes the group ownership of `myfile.txt` to `newgroup`.

## Special Permissions

- **Setuid (Set User ID)**:
  - **Usage**:
    ```bash
    chmod u+s /path/to/file
    ```
    *Example:* Allows a file to be executed with the permissions of the file owner.

- **Setgid (Set Group ID)**:
  - **Usage**:
    ```bash
    chmod g+s /path/to/directory
    ```
    *Example:* New files created within the directory inherit the group of the directory.

- **Sticky Bit**:
  - **Usage**:
    ```bash
    chmod +t /path/to/directory
    ```
    *Example:* Ensures that only the file owner can delete or rename the files in the directory.

## Viewing Special Permissions

- **View special permissions**:
  ```bash
  ls -l /path/to/file
  ```
  *Example:* Look for `s`, `S`, or `t` in the permission field:
  ```
  -rwsr-xr-x 1 root root 12345 Aug 16 08:00 /usr/bin/sudo
  drwxrwxrwt 2 root root 4096 Aug 16 08:00 /tmp
  ```

## Recursively Changing Permissions

- **Using `chmod` recursively**:
  ```bash
  chmod -R 755 /path/to/directory
  ```
  *Example:* Recursively change permissions for all files and directories within `/path/to/directory`.

- **Using `chown` recursively**:
  ```bash
  sudo chown -R newowner:newgroup /path/to/directory
  ```
  *Example:* Recursively change ownership for all files and directories within `/path/to/directory`.
