## **Introduction**

In an Unix operating system, managing users and groups can be a laborious operation, particularly when handling several users. We can automate the creation of users and groups, configure home directories, generate random passwords, and log all activities with a Bash script, which will streamline the process. You may follow along with a detailed Bash script that completes these tasks by reading this blog article.

**Prerequisites**
Before we dive into the code, ensure you have a basic understanding of the Bash shell and the permission requirements for user creation on your Linux system.

**The Bash Script**

```
#!/bin/bash

LOG_FILE="/var/log/user_management.log"
PASSWORD_FILE="/var/secure/user_passwords.csv"

# Ensure /var/secure exists and has the correct permissions
mkdir -p /var/secure
chmod 700 /var/secure
touch "$PASSWORD_FILE"
chmod 600 "$PASSWORD_FILE"

# Function to log messages
log_message() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" | tee -a "$LOG_FILE"
}

# Function to generate random passwords
generate_password() {
    openssl rand -base64 12
}

# Function to add users, groups and set up home directories
setup_user() {
    local username=$1
    local groups=$2

    # Create the user
    if ! id -u "$username" &>/dev/null; then
        password=$(generate_password)
        useradd -m -s /bin/bash "$username"
        echo "$username:$password" | chpasswd
        log_message "User $username created."

        # Store the username and password
        echo "$username,$password" >> "$PASSWORD_FILE"
        log_message "Password for $username stored."
    else
        log_message "User $username already exists."
    fi
    if ! getent group "$username" &>/dev/null; then
            groupadd "$username"
            log_message "Group $username created."
        fi
        usermod -aG "$group" "$username"
        log_message "Added $username to $group."
    # Create groups and add user to groups
    IFS=',' read -ra group_array <<< "$groups"
    for group in "${group_array[@]}"; do
        if ! getent group "$group" &>/dev/null; then
            groupadd "$group"
            log_message "Group $group created."
        fi
        usermod -aG "$group" "$username"
        log_message "Added $username to $group."
    done

    # Set up the home directory
    local home_dir="/home/$username"
    chown "$username":"$username" "$home_dir"
    chmod 700 "$home_dir"
    log_message "Home directory set up for $username  with appropriate permissions."
}


if [ $# -eq 0 ]; then
    log_message "Usage: $0 <input_file>"
    exit 1
fi

input_file=$1
log_message "Starting users and groups script."

# Read the input file and process each line
while IFS=';' read -r username groups; do
setup_user "$username" "$groups"
done < "$input_file"

log_message "Users created with password and set to groups script completed."

```

**Understanding the Script**

`#!/bin/bash`
The line #!/bin/bash at the beginning of a script is called a shebang (or hashbang). It specifies the path to the interpreter that should be used to run the script. In this case, it indicates that the script should be executed using the Bash shell located at /bin/bash.

```
# Check if script is running with sudo
if [ "$(id -u)" -ne 0 ]; then
    echo "This script must be run with sudo."
    exit 1
fi
```

`if [ "$(id -u)" -ne 0 ]; then:` Checks if the effective user ID ($(id -u)) is not equal (-ne) to 0, which is the user ID of the root user (typically indicating sudo privileges).

```
LOG_FILE="/var/log/user_management.log"
PASSWORD_FILE="/var/secure/user_passwords.csv"
mkdir -p /var/secure
chmod 700 /var/secure
touch "$PASSWORD_FILE"
chmod 600 "$PASSWORD_FILE"
```

This script makes sure that a file and directory are set up securely to store user passwords. It first determines the directories for the password and log files, and if the /var/secure directory doesn't already exist, it creates it and sets its rights so that only the owner may access it. Subsequently, it generates the password file and modifies its permissions to restrict access to only the owner. This guarantees that private password data is kept safe.

`log_message` function logs messages to the `$LOGFILE` path with date stamps

`generate_password` function creates a 12 character long random password

```
setup_user() {
    local username=$1
    local groups=$2

    # Create the user
    if ! id -u "$username" &>/dev/null; then
        password=$(generate_password)
        useradd -m -s /bin/bash "$username"
        echo "$username:$password" | chpasswd
        log_message "User $username created."

        # Store the username and password
        echo "$username,$password" >> "$PASSWORD_FILE"
        log_message "Password for $username stored."
    else
        log_message "User $username already exists."
    fi
    if ! getent group "$username" &>/dev/null; then
            groupadd "$username"
            log_message "Group $username created."
        fi
        usermod -aG "$group" "$username"
        log_message "Added $username to $group."
    # Create groups and add user to groups
    IFS=',' read -ra group_array <<< "$groups"
    for group in "${group_array[@]}"; do
        if ! getent group "$group" &>/dev/null; then
            groupadd "$group"
            log_message "Group $group created."
        fi
        usermod -aG "$group" "$username"
        log_message "Added $username to $group."
    done

    # Set up the home directory
    local home_dir="/home/$username"
    chown "$username":"$username" "$home_dir"
    chmod 700 "$home_dir"
    log_message "Home directory set up for $username  with appropriate permissions."
}
```

This script defines a function setup_user that creates a new user with specified groups. It checks if the user already exists, and if not, generates a password, creates the user, and stores the username and password in a secure file. It then creates any specified groups that do not already exist and adds the user to those groups. Finally, it sets up the user's home directory with the correct ownership and permissions.

```
if [ $# -eq 0 ]; then
    log_message "Usage: $0 <input_file>"
    exit 1
fi

input_file=$1
log_message "Starting users and groups script."

```

This piece of code determines whether any command-line arguments are supplied ($# determines the number of arguments). It reports an error message showing the right usage and quits with a status of 1, signalling an error, if none are given ($# -eq 0). It logs a message signalling the beginning of a script for managing users and groups if an input file argument is given.

```
while IFS=';' read -r username groups; do
setup_user "$username" "$groups"
done < "$input_file"

```

This script reads a file line by line, expecting each line to have a group and a username separated by a semicolon (;). It invokes the setup_user method for each line, passing the groups and username as parameters. Presumably, the setup_user function adds the user to the selected groups and creates them. Until every line in the input file has been processed, this loop keeps going.

**Running the script**

To run the script, execute it with superuser privileges (as user creation requires root access):

`sudo bash create_users.sh users.txt`
Upon execution this script will create multiple users, multiple groups and set up their home directory
