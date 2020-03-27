===============================
Reinstalling and hardening KVMs
===============================
Assumes newly provisioned, barebones, headless Debian-based KVM. Login as root with provided password.

Early system maintenance
------------------------
Make sure everything's updated (might require a reboot, but do it later)

``apt update && apt dist-upgrade && apt auto``

Install essential packages first

``apt install sudo``

Create new user, and grant ``sudo`` rights

``adduser newusername``

``usermod -aG sudo newusername``

Then re-login as "newusername", don't do things as root.

SSH hardening
-------------
Change login port, disable root login

``sudo nano /etc/ssh/sshd_config``

Port xxxx (something not-22)
PermitRootLogin no
# sends null packet every 120 secs for 720 times before timing out
# (i.e. times out after 120 s x 720 = 24 hours)
ClientAliveInterval 120
ClientAliveCountMax 720

Note: changing SSH port number is security via obscurity--not great, but not completely worthless either. Avoids getting bots that scrapes the lowest lying fruits.

Create SSH key for passwordless login

``ssh-keygen``

then paste "id_rsa.pub" from main computer into 

``nano .ssh/authorized_keys``

Restart service for changes to kick in

``sudo service ssh restart``

Kill connection and test relogging on port 22 (should fail) and xxxx (pass). If pass, reboot the KVM.

``sudo reboot now``

Additional packages to install
------------------------------
More hardening

``sudo apt install fail2ban``

Essential packages

``sudo apt install apt-listchanges curl htop nano rsync``
