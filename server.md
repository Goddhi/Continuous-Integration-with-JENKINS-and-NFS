

<h2>CONTINUOS INTEGRATION with JENKINS and NFS</h2>


How to fix HTTP ERROR 403 No valid crumb was included in the request in JENKINS
SOLUTION
There is an option in "Manage Jenkins" the "Global Security Settings" that "Enables the Compatibilty Mode for proxies". This helped with my issue.
Manage Jenkins > Configure Global Settings > CSRF Protection > Enable proxy compatibility











sudo mount -t nfs -o rw,nosuid 172.31.19.51:/mnt/apps /var/www
172.31.19.51:/mnt/apps /var/www nfs defaults 0 0

## 29a54e1aeb544ec6a63e678a981a44f6