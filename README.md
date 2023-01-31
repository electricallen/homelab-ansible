# NextCloud, Bitwarden, Heimdall, and Traefik Containers in Ansible

An ansible playbook to install and configure a homelab using docker conatiners on an Ubuntu host

## Services included

* [Nextcloud](https://nextcloud.com) - Cloud storage and more
* [Bitwarden_rs](https://github.com/dani-garcia/bitwarden_rs) - Password manager. A community made server implementation of [Bitwarden](https://bitwarden.com)
* [Heimdall](https://github.com/linuxserver/Heimdall) - Application dashboard
* [Traefik](https://traefik.io) - Reverse proxy, HTTPS certificate management, and more
* [Kopia](https://github.com/kopia/kopia) - Automatic backup to cloud or local storage

## What you need

* A server capable of running Ubuntu either as a VM or directly (~16 GB storage, ~1 GB RAM) - the Ansible target node
* A domain name
* A router capable of port forwarding, DDNS, and split DNS override (pfSense is a fantastic option)
* A computer to be used as an Ansible Control node. This must be linux or WSL on a Windows machine
* Optional - A backblaze b2 bucket for automatic backups. Many storage locations can be configured manually through kopia, but only b2 is automated here

## How to Use

1. Configure your settings in `defaults/main.yml` within the directory with this README.md
2. Install Ubuntu 22.04 and OpenSSH on target node (20.04 and 18.04 are also tested and supported - newer/older versions may or may not work)
3. Set up port forwarding/routing

   Router
  
    * Port forward http and https to the target node (typically :80 and :443)
    * Set up split dns override to route the hostnames of all of your services to the docker host

   Domain registrar

    * Set up routing for your domain - an A+ record with wildcard (*) for hostname will work

4. Ensure ssh keys are set up between ansible control and target nodes

    * Run these commands from anisble control node

    ```sh
    sudo apt install openssh.server
    ssh-keygen
    ssh-copy-id <user>@<target_node_ip_or_hostname>
    ```

5. `cd` to the directory with this README.md
6. Install Ansible on control node

    ```sh
    sudo apt update
    sudo apt install ansible
    ansible-galaxy install -r requirements.yml
    ```

7. Run the ansible playbook from this directory with:

    ```sh
    ansible-playbook homelab.yml -i hosts.yml
    ```

8. Once the playbook is done, check to see if your routing works by typing `traefik.yourdomain.xyz` into a browser
9. It may take a few minutes for traefik to install HTTPS certificates from Let's Encrypt

    * You can check the status of certificates by inspecting `~/traefik/letsencrypt/acme.json` on the docker host
10. Configure Nextcloud

    * Check the overview and security screens in settings when logged in as an admin
    * If you get a warning about missing indicies, run `docker-compose exec --user www-data nextcloud php occ db:add-missing-indices` on the target node

## Configure Backups

* Log into the Kopia web UI to configure storage, snapshots, and policies
* Shell scipts exist to backup databases prior to running a snapshot:
  * `~/bitwarden/bitwarden/backup.sh` will backup vaultwarden's SQLite DB
  * `~/nextcloud/db/backup.sh` will backup nextcloud's MySQL DB
* The docker host `/` directory is bind mounted to the Kopia container at `/data`. For example, to backup vaultwarden:
  * Use `/data/home/<yourUsername>/bitwarden` as the directory
* To access a directory from the Kopia UI, prepend `/data`. For example, to snapshot the bitwarden directory, use `/data/home/{{ username }}/bitwarden`

### Nextcloud

* The nextcloud directory contain everything needed to properly restore nextcloud from backups
* A backup script, `~/nextcloud/db/backup.sh` runs periodically to backup the contents of the database
* A restore script is available at `~/nextcloud/db/restore.sh`

## Restore backups

Backup restoration steps here are broken up into two type - standard and disaster recovery

### Standard Backups

These steps outline how to recover data on a running and configured host

1. Stop all compose stacks you wish to restore except traefik and Kopia
1. Enable read/write access to Kopia:
    * Edit `~/kopia/docker-compose.yml`
    * Change `/:/data:ro` to `/:/data:rw` in the `volumes` section
    * `docker-compose up -d`
1. Restore snapshot(s) through Kopia web UI, remembering to prepend `/data/home/<yourUsername>` to paths
1. To restore bitwarden run `sudo ~/bitwarden/restore.sh`
1. If Nextcloud needs to be restored:

    * Run `~/nextcloud/db/restore.sh` - it may take a bit for the DB container to accept connections
    * Fingerprint to fix issues with sync clients:

    ```sh
    docker exec --user www-data nextcloud php occ maintenance:mode --on
    docker exec --user www-data nextcloud php occ maintenance:data-fingerprint
    docker exec --user www-data nextcloud php occ maintenance:mode --off
    ```

1. If you get an `Internal Server Error` on the Nextcloud webpage, see the [Nextcloud Troubleshooting](#nextcloud-troubleshooting) section

### Disaster Recover

These steps outline how to restore backups to a new host

1. Run the `homelab.yml` playbook to deploy all services you use
1. Stop all compose stacks except traefik and Kopia
1. Enable read/write access to Kopia:
    * Edit `~/kopia/docker-compose.yml`
    * Change `/:/data:ro` to `/:/data:rw` in the `volumes` section
    * `docker-compose up -d`
1. Restore backups through Kopia web UI, remembering to prepend `/data/home/<yourUsername>` to paths
1. Run `homelab.yml` again
1. To restore bitwarden run `sudo ~/bitwarden/restore.sh`
1. If Nextcloud needs to be restored:

    * Run `~/nextcloud/db/restore.sh` - it may take a bit for the DB container to accept connections
    * Fingerprint to fix issues with sync clients:

    ```sh
    docker exec --user www-data nextcloud php occ maintenance:mode --on
    docker exec --user www-data nextcloud php occ maintenance:data-fingerprint
    docker exec --user www-data nextcloud php occ maintenance:mode --off
    ```

1. If you get an `Internal Server Error` on the Nextcloud webpage, see the [Nextcloud Troubleshooting](#nextcloud-troubleshooting) section

### Nextcloud Troubleshooting

Nextcloud backups can fail to restore for a number of reasons:

1. File and folder permissions

    * Nextcloud will fail to run if the `~/nextcloud/nextcloud` files and folders don't have proper permissions
    * Check to ensure you restored from Kopia with overwrite file permissions enabled

1. Database Issues
    * Sometimes, the database `nextcloud` user's hashed password doesn't match the nextcloud container's password (can happen when repetedly restarting containers)
    * You'll see errors in the compose log that look like this:

    ```sh
    [Warning] Aborted connection 8 to db: 'unconnected' user: 'unauthenticated' host: '192.168.96.5' (This connection closed normally without authentication)
    ```

    * Take the compose stack down, prune volumes, restart compose stack, then restore the database

    ```sh
    docker-compose down
    docker volume prune
    docker-compose up
    # wait for the db container to start
    ./db/restore.sh
    ```

## FAQs

### How can I add a new service to this playbook?

These are the steps I reccomend using to add services:

1. Build and test your `docker-compose.yml` file, verifying it works as expected
1. Add the service to the `services` list in `default/main.yml`
1. Add any settings for the `docker-compose.yml` file to a new section in `default/main.yml`
1. Update your `docker-compose.yml` file with the new settings you added, using [Jinja2](https://palletsprojects.com/p/jinja/) templating
1. Add the templated compose file at `templates/<serviceName>/docker-compose.yml.j2`
1. If you need any additional directories or files add them at `templates/<serviceName>/`
    * This will create folders and files at `~/<serviceName>/` on the target node
    * Note all template files will need the `.j2` extension appended (EG `docker-compose.yml` becomes `docker-compose.yml.j2`)

This is all that is needed for most applications. However, if you have any tasks that need to run after installation add them as a separate ansible task at `tasks/post_install/<serviceName>.yml`

### Why ansible?

Ansible enables rapid redeployment of services in the event of hardware or software failure.
Since Ansible is state-based, it can also be used to fix configs that were inadvertantly edited. It's also fun to write!
