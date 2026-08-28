# ProxmoxVE Host User Shell Auto Login

A simple bash script to add the current user to the shell auto login like the root user does.

The file for the login function on the host shell is `/usr/share/perl5/PVE/API2/Nodes.pm`. The source file is located at [proxmox/pve-manager/PVE/API2/Nodes.pm](https://github.com/proxmox/pve-manager/blob/master/PVE/API2/Nodes.pm) on GitHub.

> [!Important]
> **The script needs to be run after every `apt upgrade` of `pve-manager` since the file will be overwritten.** You'll notice the login prompt coming back after an update, just login back in and run it again.

## Instructions

Two options are available : via script or manually. Choose what you prefer.

### Running the script

Running the script is easier and faster.

1. Download the script : `wget file.sh`
2. Make it executable : `chmod +x file.sh`
3. Run the script : `./file.sh`

### Manually editing the file

If you prefer editing the file manually, do the following.

1. Open the file `/usr/share/perl5/PVE/API2/Nodes.pm` in nano or prefered editor
   Ex.: `sudo nano /usr/share/perl5/PVE/API2/Nodes.pm`
2. Search the following line : `sub get_shell_command` (around line 1159)
   Use the `CTRL + F` shortcut to search in nano. Use `CTRL + C` to show current line.
3. The function `get_shell_command` looks like this  :
    ```bash
    sub get_shell_command {
        my ($user, $shellcmd, $args, $tunnel_cmd) = @_;

        my $is_ssh_tunneling = defined($tunnel_cmd) && scalar(@$tunnel_cmd);

        my $cmd;
        if ($user eq 'root@pam') {
            if (defined($shellcmd) && exists($shell_cmd_map->{$shellcmd})) {
                my $def = $shell_cmd_map->{$shellcmd};

                if ($is_ssh_tunneling && $shellcmd eq 'login') {
                    # stop-gap to avoid running into a racy bug with nested login, i.e. first from SSH
                    # second would be this command here, likely related to vhangup.
                    $cmd = [];
                } else {
                    $cmd = [$def->{cmd}->@*]; # clone
                }

                if (defined($args) && $def->{allow_args}) {
                    push @$cmd, split("\0", $args);
                }
            } elsif ($is_ssh_tunneling) {
                $cmd = []; # SSH logs us already in as root, and we must not nest login (vhangup).
            } else {
                $cmd = ['/bin/login', '-f', 'root'];
            }
        } else {
            # non-root must always login for now, we do not have a superuser role!
            $cmd = ['/bin/login'];
        }

        return $is_ssh_tunneling ? [$tunnel_cmd->@*, $cmd->@*] : $cmd;
    }
    ```
4. Add a `elif` statement to check for your user :
    ```bash
    sub get_shell_command {
        #...
        if ($user eq 'root@pam') {
            #...
            } else {
                $cmd = ['/bin/login', '-f', 'root'];
            }
        } elsif ($user eq 'noah@pam') {     # change 'noah' to your username
            $cmd = ['/bin/login', '-f', 'noah'];    # change here also
        } else {
            # non-root must always login for now, we do not have a superuser role!
            $cmd = ['/bin/login'];
        }
        #...
    }
    ```
5. Search the following line : `if (defined($param->{cmd}) && $param->{cmd}` (around line 1535)
   Use the `CTRL + F` shortcut to search in nano. Use `CTRL + C` to show current line.
6. You'll find this :
   ```bash
   if (defined($param->{cmd}) && $param->{cmd} ne 'login' && $user ne 'root@pam') {
        raise_perm_exc('user != root@pam');
    }
   ```
7. Edit with your user as the following :
   ```bash
   if (defined($param->{cmd}) && $param->{cmd} ne 'login' && ($user ne 'root@pam' && &user ne 'noah@pam')) {
        raise_perm_exc('user != root@pam');
    }
   ```
   If you have multiple users, add them to the AND :
   ```bash
   ($user ne 'root@pam' && &user ne 'noah@pam' && &user ne 'bob@pam')
   ```
8. Save the file. `CTRL + X` then `Enter` in nano.
9. Restart the daemon and proxy : `sudo systemctl restart pvedaemon pveproxy`
    The interface might disconnect for a few seconds.