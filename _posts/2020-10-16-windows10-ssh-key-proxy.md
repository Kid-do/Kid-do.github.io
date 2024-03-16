---
title: "SSH Key-Based Authentication through a Proxy on a Windows 10 Client"
categories:
  - blog
---

I wanted to connect to one of my University's Linux computers to compile and run some code. However, in order to do this, I needed to connect through an SSH proxy. To simplify this process, I also wanted to set up SSH key-based authentication. This post will document how I set this up on a Windows 10 client.

Here is what we are trying to do diagrammatically:

````
+---------------------+      +----------------+      +------------------+
|                     |      |                |      |                  |
|  Windows 10 Client  ------->  Proxy Server  ------->  Remote Machine  |
|                     |      |                |      |                  |
+---------------------+      +----------------+      +------------------+
````

Firstly, start by opening or creating the OpenSSH config file, on Windows this will be in the .ssh folder in your user directory. For me it is `C:\Users\Ben\.ssh\config`. Add the following lines to the file:

````
Host proxy
  HostName <proxy_server_address>
  User <proxy_server_user>
````

This configuration allows you to access a remote SSH server without having to type in your username and the address each time. Try typing `ssh proxy` into PowerShell or the Command Prompt, then type your password when you are prompted for it and you should be logged into the server.

Now, on the Windows 10 client, generate a SSH key pair with the command:

```
ssh-keygen -t rsa
```

Normally, if you were using Linux, you'd be able to use the `ssh-copy-id` command to copy the public key you have just generated to the remote machine, however, Windows 10's OpenSSH implementation does not yet have this feature, so you need to use this command which will do the same thing:

````
type $env:USERPROFILE\.ssh\id_rsa.pub | ssh proxy "cat >> .ssh/authorized_keys"
````

Now you should be able to access the proxy server by typing `ssh proxy` without entering a password.

While you are SSH'd into the proxy server, you now need to copy the public key to the target machine using the SCP command:

````
scp .ssh/authorized_keys <user>@<address>:.ssh
````

Finally, going back to the SSH config file we created earlier, add the following lines:

````
Host target
  HostName <target_server_address>
  User <target_server_user>
  ProxyCommand ssh.exe -W %h:%p proxy
````

This allows you to simply type `ssh target` and you are automatically proxied through the proxy server to the target machine.

Now you are done, I hope this was helpful to you. Please comment if you have any issues. This setup will also work with VS Code and the Remote-SSH extension to allow you to develop on the remote machine as if it were your local machine too.

Thanks for reading!

Sources:

[Windows 10 ssh-copy-id](https://www.chrisjhart.com/Windows-10-ssh-copy-id/)

[Windows 10 SSH ProxyCommand](https://github.com/PowerShell/Win32-OpenSSH/issues/1172#issuecomment-449582869)