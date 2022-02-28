# PSTF Holding Website
This work is licensed under the [Apache License, Version 2.0](http://www.apache.org/licenses/LICENSE-2.0).

## 1. Deployment

This is managed using git hooks. You'll need the server in your ```.ssh/config``` file, and the remote:

    git remote add production user@server:pstf-holding.git

You can only deploy the main branch. To do so, run the following command:

    git push production main

i.e. you are pushing main to the production remote. This will run the hooks, and restart ```nginx``` for you.

If you want to execute the hooks again without changing the code, you'll need an empty commit, then push to production:

    git commit --allow-empty -m "trigger deploy"
    git push production main

## 2. Server configuration

Requires a bare git repo on the host, called `pstf-holding.git`. This is created with
`git init --bare pstf-holding.git` in the home directory.

Create the working tree directory where nginx will serve the files from and fix the permissions:

    sudo mkdir /var/www/pstf-holding
    sudo chown www-data:www-data /var/www/pstf-holding
    sudo chmod -R 775 /var/www
    sudo usermod -a -G www-data user

The post receive hook then needs to be created on the host, by copying `deploy/hooks/post-receive` to
`~/pstf-holding.git/hooks/post-receive` and `chmod +x hooks/post-receive` on the host. This will allow the script to
run on deployment and copy the code to the correct work tree.

Run a code deploy from your local machine to receive the source tree for the first time.

    git push production main

N.B. if your production server is running an old version of `git` it may not know about the `main` branch being default. The error looks like:

    remote: Main ref received.  Deploying main branch to production...
    remote: fatal: You are on a branch yet to be born

Fix that by checking out `main` on the host server with e.g.:

    git --work-tree=/var/www/pstf-holding --git-dir=/home/user/pstf-holding.git checkout -f main

After the first deploy, replace the copied hook with a symlink to the checked out hook (so you can update the hook):

    ln -sf /var/www/pstf-holding/deploy/hooks/post-receive hooks/post-receive
    chmod +x hooks/post-receive
    chmod +x /var/www/pstf-holding/deploy/hooks/post-receive

## 3. SSL Certificates

The nginx config in `deploy/nginx` expects an SSL certificate - this should be created via LetsEncrypt / certbot.

For your SSL certificate to validate, the site must be accessible via nginx; create the symlinks to the temporary
nginx config:

    sudo ln -s /var/www/pstf-holding/deploy/nginx/init.jcs_com_io_org /etc/nginx/sites-available/jcs_com_io_org
    sudo ln -s /etc/nginx/sites-available/jcs_com_io_org /etc/nginx/sites-enabled/jcs_com_io_org
    sudo nginx -s reload

Run certbot:

    sudo certbot                                                                           # And follow the instructions

It will make changes to the initialisation config, discard these changes so the source tree is clean.

    git --work-tree=/var/www/pstf-holding --git-dir=/home/cloo/pstf-holding.git reset --hard

Then switch over to the production nginx config:
    sudo ln -sf /var/www/pstf-holding/deploy/nginx/jcs_com_io_org /etc/nginx/sites-available/jcs_com_io_org
    sudo nginx -s reload
