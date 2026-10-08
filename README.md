# ProxmoxVE User Shell Auto Login

A simple bash script to add the current user to the shell auto login like the root user does. This allows the user to be automatically logged in to the shell without entering credentials.

### Before
![Before enabling auto login](https://raw.githubusercontent.com/noahbleau/PVE-Shell-Auto-Login/refs/heads/main/img/before.png)

### After
![After enabling auto login](https://raw.githubusercontent.com/noahbleau/PVE-Shell-Auto-Login/refs/heads/main/img/after.png)

The file for the login function on the host shell is `/usr/share/perl5/PVE/API2/Nodes.pm`. *The source for this file is available here : [proxmox/pve-manager/PVE/API2/Nodes.pm](https://github.com/proxmox/pve-manager/blob/master/PVE/API2/Nodes.pm).*

> [!Important]
> **The script needs to be run after every `apt upgrade` of `pve-manager` since the file will be overwritten.** You'll notice the login prompt coming back after an update, just log back in and run these steps again.

> [!Warning]
> **Modifying system files can potentially break your Proxmox installation.** Proceed with caution and make sure to have a backup before proceeding.

## Instructions

Two options are available : via script or manually. Choose what you prefer.

### Running the script

Running the script is easier and faster.

1. Download the script : `wget https://raw.githubusercontent.com/noahbleau/PVE-Shell-Auto-Login/refs/heads/script/pve-auto-login.sh`

   > **This downloads from the `script` branch.**

3. Make it executable : `chmod +x pve-auto-login.sh`
4. Run the script : `./pve-auto-login.sh`

### Manually editing the file

If you prefer editing the file manually, do the following.

1. Open the file `/usr/share/perl5/PVE/API2/Nodes.pm` in nano or your preferred editor.

   ```bash
   sudo nano /usr/share/perl5/PVE/API2/Nodes.pm
   ```

2. Search for the following line in the file: `sub get_shell_command` *(around line 1159)*.

   > **Tip:** Use the `CTRL + F` shortcut to search in nano and use `CTRL + C` to show the current line.

3. Add a `elif` statement inside the function to check for your user.

   It will be between the `if ($user eq 'root@pam')` and the final `else` :

    ```bash
    sub get_shell_command {
        if ($user eq 'root@pam') {
            # ...
        } elsif ($user eq 'noah@pam') {     # change 'noah' to your username
            $cmd = ['/bin/login', '-f', 'noah'];    # change here also
        } else {
            # ...
        }
    }
    ```

4. Search for the following line in the same file : `if (defined($param->{cmd}) && $param->{cmd}` *(around line 1535)*.

   > **Tip:** Use the `CTRL + F` shortcut to search in nano and use `CTRL + C` to show the current line.

5. Edit the condition to include your user as well as root.

   **Please note the `&user` infront of your username, the root user has a different `$user`.**
   ```bash
   if (defined($param->{cmd}) && $param->{cmd} ne 'login' && ($user ne 'root@pam' && &user ne 'noah@pam')) {
        raise_perm_exc('user != root@pam');
    }
   ```

   > **Note:** Please make sure to include the `()` around the users in the condition.

   If you have multiple users, add them to the AND as the example below:

   ```bash
   ($user ne 'root@pam' && $user ne 'noah@pam' && $user ne 'bob@pam')
   ```
6. Save the file. Use `CTRL + X` then `Enter` for nano.

7.  Restart the daemon and proxy :

    ```bash
    sudo systemctl restart pvedaemon pveproxy
    ```

    The interface might disconnect for a few seconds.
