# nanobsd-scripts

Enhanced config management scripts for NanoBSD.

## `/root/save_cfg`

This script is a patched version of the default `/root/save_cfg` that adds support for symbolic link backup.

## `/root/commit_cfg`

This script commits the content of `/cfg/` slice to a git and push it to multiple remotes. It also provide an encryption mechanism for sensitive files based on openssl with aes-256.

It also allows restoring `/cfg` content by pulling from the remote gits. However, it does not update in memory `/etc`.

No synchronization should be required between the remote gits as long as only this script is used to push to the remotes.

### Requirements

The script requires the following packages to be installed:

* bash
* rsync
* git

### Setup

It requires an alternate slice mounted to `/cfg-git` to store the git and other data (encryption key, encrypted file hashes, encrypted file list, ...).

`NANO_DATASIZE` and the function `setup_nanobsd_cfg_git` must be added in the NanoBSD compilation config file:

```
NANO_DATASIZE=1310720 # or whatever size you see fit

setup_nanobsd_cfg_git () (
    echo "/dev/${NANO_DRIVE}${NANO_SLICE_DATA} /cfg-git ufs rw,noauto 2 2" >> ${NANO_WORLDDIR}/etc/fstab
    mkdir -p ${NANO_WORLDDIR}/cfg-git
)

customize_cmd setup_nanobsd_cfg_git
```

### Example

* Initialize the git repository:

```
# /root/commit_cfg init -k /etc/nanodeployer_decryption.key -u "root@$(hostname)" -m "<my_email_address>"
```

* Add a git remote:

```
# /root/commit_cfg add-remote -n <remote_name> -u https://user:pwd@remote.url/
```

* List remotes:

```
# /root/commit_cfg list-remotes
```

* Remove a git remote:

```
# /root/commit_cfg del-remote -n <remote_name>
```

* Mark some files for encryption:

```
# /root/commit_cfg mark /etc/nanodeployer_decryption.key
# /root/commit_cfg mark /etc/rc.conf
# /root/commit_cfg mark /etc/ssh/ssh_host_rsa_key
# /root/commit_cfg mark /etc/ssh/ssh_host_ecdsa_key
# /root/commit_cfg mark /etc/ssh/ssh_host_ed25519_key
```

* List marked files:

```
# /root/commit_cfg list-marks
```

* Unmark a file:

```
# /root/commit_cfg unmark /etc/rc.conf
```

* Commit and push `/cfg` data:

```
# /root/commit_cfg push
```

* Pull and restore `/cfg`:

```
# /root/commit_cfg pull
```

## `/usr/local/rc.d/nanodeployer`

The `nanodeployer` rc script is used as a simple way to manage NanoBSD provisionning. It pulls config from multiple remote gits using the `commit_cfg` script and can reboot if any change arises. It also updates the NanoBSD image using the `update` script and can also reboot if needed.

### Requirements

The script requires the additional following packages to be installed:

* bash
* curl

### Setup

On a running NanoBSD, the current build version is obtained by looking at the file `/nanobsd.version`. This file is created by adding the following function to the compilation config file:

```
add_nanobsd_version_file () (
    version=$(date "+%a %b %d %T %Y")
    echo "nanobsd.version: ${version}"
    echo "${version}" > ${NANO_WORLDDIR}/nanobsd.version
)

customize_cmd add_nanobsd_version_file
```

### Config

The rc script fetches the following variables from the `rc.conf` file:

* `nanodeployer_enable`: enable/disable the service (defaults to `"NO"`)
* `nanodeployer_remotes`: list of git remotes to fetch config from (defaults to `""`). Each element of this space separated list must follow the format: `<remote_name>::<git_remote_url>`.
* `nanodeployer_git_email`: email to set in git config (defaults to `"root@$(hostname -f)"`)
* `nanodeployer_git_username`: username to set in git config (defaults to `"root@$(hostname -f)"`)
* `nanodeployer_reboot_on_changes`: reboot when changes are found in pulled config or when image is updated (defaults to `"NO"`)
* `nanodeployer_image_urls`: urls from where to fetch updated images (defaults to `""`). Each url is separated by a space.
* `nanodeployer_decryption_key_file`: path to the file containing the encryption/decryption key (defaults to `"/etc/nanodeployer_decryption.key"`)

example:

```
nanodeployer_enable="YES"
nanodeployer_remotes="nanodeployer01::https://test1:pwd1@nanodeployer01.example.net:8443/git/testing nanodeployer02::https://test1:pwd1@nanodeployer02.example.net:8443/git/testing nanodeployer03::https://test1:pwd1@nanodeployer03.example.net:8443/git/testing"
nanodeployer_reboot_on_changes="YES"
nanodeployer_image_urls="https://test1:pwd1@nanodeployer01.example.net:8443/images/testing https://test1:pwd1@nanodeployer02.example.net:8443/images/testing https://test1:pwd1@nanodeployer03.example.net:8443/images/testing"
nanodeployer_git_email="email@example.com"
```

### Deployment servers

The setup of a deployment server is very simple:

* create directories for git and the updated image:

```
mkdir -p /srv/nanodeployer/images/testing /srv/nanodeployer/git/testing
```

* initialize a bare git repository:

```
git init --bare /srv/nanodeployer/git/testing
chown -R www-data:www-data /srv/nanodeployer/git/testing
```

* create an htpasswd file to protect the git and the image:

```
htpasswd -c /srv/nanodeployer/testing.htpasswd <user>
chown root:www-data /srv/nanodeployer/testing.htpasswd
chmod 0640 /srv/nanodeployer/testing.htpasswd
```

* nginx configuration:

```
server {
    listen 443 ssl;
    server_name ~^nanodeployer0[1-3]\.example\.net$;

    ssl on;
    ssl_certificate           /etc/nginx/ssl/nanodeployer01-3.example.net.crt.pem;
    ssl_certificate_key       /etc/nginx/ssl/nanodeployer01-3.example.net.key.pem;

    access_log            /var/log/nginx/nanodeployer.access.log;
    error_log             /var/log/nginx/nanodeployer.error.log;

    root /srv/nanodeployer/images;

    location ~ /git(/testing/.*) {
        auth_basic "restricted access";
        auth_basic_user_file /srv/nanodeployer/testing.htpasswd;

        client_max_body_size 0;
        fastcgi_pass         unix:/var/run/fcgiwrap.socket;
        include              fastcgi_params;
        fastcgi_param        SCRIPT_FILENAME                /usr/lib/git-core/git-http-backend;
        fastcgi_param        GIT_HTTP_EXPORT_ALL            "";
        fastcgi_param        GIT_PROJECT_ROOT               /srv/nanodeployer/git;
        fastcgi_param        PATH_INFO                      $1;
        fastcgi_param        REMOTE_USER                    $remote_user;
    }

    location ~ /images/testing/ {
        auth_basic "restricted access";
        auth_basic_user_file /srv/nanodeployer/testing.htpasswd;

        root /srv/nanodeployer;
        try_files $uri $uri/ =404;
    }

    location / {
        return 403;
    }

    location = /robots.txt {
        allow all;
        log_not_found off;
        access_log off;
    }

    location ~ /\. {
        deny all;
        access_log off;
        log_not_found off;
    }
}
```

* upload NanoBSD slice image in `/srv/nanodeployer/images/testing/image.gz` and set ownership to `root:www-data` and permissions to `0640`.

* setup version file content to the date found in `/usr/obj/nanobsd/_.cust.add_nanobsd_version_file` on building machine:

```
echo 'Tue Nov 13 10:42:48 2018' > /srv/nanodeployer/images/testing/version
chown root:www-data /srv/nanodeployer/images/testing/version
chmod 0644 /srv/nanodeployer/images/testing/version
```

Updating an image is as simple as replacing the `image.gz` file in the right directory and updating the `version` file with the build date.

Be careful to setup the content of the `version` file accordingly to the date displayed in `/usr/obj/nanobsd/_.cust.add_nanobsd_version_file`. Indeed, if the date in `version` is greater than the date in the `/nanobsd.version` file in the image, the NanoBSD will upgrade and reboot indefinitely.

### Boot strategy

Two ways are possible for the rc script invocation:

* as a classic rc script. This approach is the most natural. However, a problem arises from the fact that an image upgrade can block the boot process for a long time. For some setups, it can be problematic if some services are not available during the boot upgrade process.
* as a crontab entry with the `@reboot` keyword. This way, the boot process is not impacted and the image upgrade takes place in the background.

This is an example of a possible crontab entry:

```
@reboot root /etc/local/rc.d/nanodeployer start 2>&1 | /usr/bin/logger -t nanodeployer -p local0.notice
```

If you choose to use it as a classic rc script, don't forget to remove the following line from the rc script. Otherwise, it will not be scheduled at all.

```
# KEYWORD: nostart
```

### Git over SSH vs git over HTTPS

When using this deployment script, git over SSH is problematic on some key security points. Indeed, unless you ensure that your ssh remote fingerprints will NEVER change or setup an additional fingerprint deployment mechanism, you will need to add `StrictHostKeyChecking=no` for the remotes in the root ssh config. The security implication is that in the case of a man in the middle attack an attacker can impersonate the remotes and intercept a git push, add some malevolent config and wait for a pull (during a reboot for example). Some will say that it's a little far fetched but it's a real security breach.

This problem doesn't occur when using git over HTTPS (when certificates are properly managed). Moreover, the NanoBSD image is already fetched over HTTPS. So, since a part of the job has already to be done, why add an other communication canal and not reuse the one already in place ?
