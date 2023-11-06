# Grub

## Password protecting menu entries
Password protecting Grub entries has some... interesting things happening.
We can piece together working instructions from <https://github.com/trimstray/the-practical-linux-hardening-guide/wiki/Bootloader-and-Partitions> and <https://help.ubuntu.com/community/Grub2/Passwords>

Along with some clever file piping, we can do some neat things.
The trick is that if you modify `00_header` as recommended in the Ubuntu link, you need the extra `cat << EOF` and `EOF` lines because of the way the files generate the boot config.
By adding the files to a different file (like `40_custom`), the infrastructure is in place to generate the commands necessary to password protect the grub entries _without needing the heredoc in the config file_

You may want to backup the `/etc/grub.d/40_custom` file before working below in case a mistake is made

### 1. Generate the password hash

We can use the `grub-mkpasswd-pbkdf2` command to generate a password hash instead of putting a plaintext password in config files (which is a bad plan(tm)).
This command has two major issues:
1. It is interactive
1. The prompts and the output go to standard output (which makes getting just the hash gnarly, and redirecting the output do weird things).

Let's start by generating the password hash and dumping the output to a file:

```bash
grub-mkpasswd-pbkdf2 | tee passwordhash
```

The `tee` command takes the standard input and echos it to standard output AND the file specified, which is helpful.

If you look at the passwordhash file, you see it has:
```text
Enter password:
Reenter password
PBKDF2 hash of your password is grub.pbkdf2.sha512.10000.<VOMIT>
```

We need just the last bit (that starts `grub.pbk`) for our file.
Our goal is to *append* the following to the file `/etc/grub.d/40_custom`:

```bash
set superusers="admin"
password_pbkdf2 admin grub.pbkdf2.sha512.10000.<VOMIT>
```

We can do this in a few ways, let's explore two options

#### Option 1: Getting the contents into the file, then adding the rest

We can parse out just the password hash knowing something about the structure of the file and using `awk` or `cut` and the `tail` command:

```bash
tail -n1 passwordhash | awk '{print $NF}'
```

This command SHOULD just print the big vomit bit at the end of the passwordhash file.
We can append it to the `/etc/grub.d/40_custom` file using `tee` again:

:warning: Make *sure* you have the `-a` in the tee command below before hitting enter; otherwise you will overwrite the file which we do not want :warning:

```bash
tail -n1 passwordhash | awk '{print $NF}' | sudo tee -a /etc/grub.d/40_custom
```

Then modify the file using `sudo nano /etc/grub.d/40_custom` to add the rest of the stuff you want.

#### Option 2: Bash File Redirection Shenanigans

We do have the power to do really cool things with using something called a heredoc (which is what the original instructions want you to use in the config file).

You can add the correct file contents using the following (multiline) command.

:warning: Again, the `-a` is SUPER IMPORTANT

```bash
sudo tee -a /etc/grub.d/40_custom << EOF
set superusers="admin"
password_pbkdf2 admin `tail -n1 passwordhash | awk '{print $NF}'`
EOF
```

Check the contents of `/etc/grub.d/40_custom` to make sure it looks right

### Update the bootloader

Finally, we can update the bootloader by running `sudo update-grub`.  
Try rebooting your VM and see if you can edit a menu entry (select it and press `e`); you should be prompted for a username and password

