# NextCloud, Bitwarden, Heimdall, and Traefik Containers in Ansible

An ansible playbook to install and configure a homelab using docker conatiners on an Ubuntu host

## Services included
* [Nextcloud](https://nextcloud.com) - Cloud storage and more
* [Bitwarden_rs](https://github.com/dani-garcia/bitwarden_rs) - Password manager. A community made server implementation of [Bitwarden](https://bitwarden.com)
* [Heimdall](https://github.com/linuxserver/Heimdall) - Application dashboard
* [Traefik](https://traefik.io) - Reverse proxy, HTTPS certificate management, and more 

## What you need
* A server capable of running Ubuntu either as a VM or directly (~16 GB storage, ~1 GB RAM) - the Ansible target node
* A domain name 
* A router capable of port forwarding, DDNS, and split DNS override (pfSense is a fantastic option)
* A computer to be used as an Ansible Control node. This must be linux. I use WSL on a Windows 10 machine

## How to Use

1. Configure your settings in `defaults/main.yml` within the directory with this README.md
2. Install Ubuntu 20.04 and OpenSSH on target node (18.04 is also tested and supported - newer/older versions may or may not work)
3. Set up port forwarding/routing

   Router
        
    - Port forward http and https to the target node (typically :80 and :443)
    - Set up split dns override to route the hostnames of all of your services to the docker host

   Domain registrar

    - Set up routing for your domain. I used an A+ record with wildcard (*) for hostname

4. Ensure ssh keys are set up between ansible control and target nodes

    * Run these commands from anisble control node
    ``` 
        sudo apt install openssh.server
        ssh-keygen
        ssh-copy-id <user>@<target_node_ip_or_hostname>
    ```
5. `cd` to the directory with this README.md
4. Install Ansible on control node
    ```
    sudo apt update
    sudo apt install ansible
    ansible-galaxy install -r requirements.yml
    ```
7. Run the ansible playbook from this directory with:
        ```
        ansible-playbook homelab.yml -i hosts.yml
        ```
8. Once the playbook is done, check to see if your routing works by typing traefik.yourdomain.xyz into a browser
9. It may take a few minutes for traefik to install an HTTPS cert from Let's Encrypt

    * You can check the status of this certificate by inspecting the contents of `~/traefik/letsencrypt/acme.json` on the docker host
10. Configure Nextcloud
    * Create user accounts for your users at the URL created in 
    * Log into clients (windows, android, etc) with the user account credentials you created
    * Check the overview and security screens in settings when logged in as an admin
    * If you get a warning about missing indicies, run `docker-compose exec --user www-data nextcloud php occ db:add-missing-indices` from `~/nextcloud` on the target node

## FAQs
### Why only these services? What If I want to do some advanced thing not included here?
This won't be perfect for anyone except myself. However, leveraging the power of ansible and `docker-compose`,
I believe it should be easily adaptable to fit your own homelab automation.

### Why ansible? 
Ansible enables rapid redeployment of services in the event of hardware or software failure. 
Since Ansible is state-based, it can also be used to fix configs that were inadvertantly edited. Its also fun to write!
