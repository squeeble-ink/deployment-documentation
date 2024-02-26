# deployment-documentation
Documentation on how Squeeble sets up a deployment pipeline.
This documentation is published online for transparency.

We think transparency is importent since Squeeble is a privacy first company.

## Deploy keys

Setting up your deploy key so the workflows can clone the repository.
Do this per server instance so when one server is compromised we can delete the deployment key linked to that server.  
This required for the following types of repositories:
-  Private with `write` and/or `read` access
-  Public with `write` access

Base documentation comes from [GitHub Docs](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/managing-deploy-keys#set-up-deploy-keys)

1.  The first step is to generate a new ssh key on the server.  
    So ssh in there on the correct account
    ```
    cd ~/.ssh/
    ssh-keygen -t ed25519
    ```
    `ssh-keygen` requests a file name.
    To not lose track of the ssh key and repository link we recommend you to give it the repository name.
    Otherwise you can name it and have a secure note that links them together. 
1.  In the sidebar, click **Deploy Keys**.
1.  Click **Add deploy key**.
1.  In the "Title" field, provide a title of the key name or alias and server identification (if 
1.  In the "Key" field, paste your public key.  
    To get the public key you can do
    ```
    cat <deploy-key-name>.pub
    ```
1.  Click **Add key**

## Secrets and veriables 

1.  In the sidebar, click **Secrets and veriables**.
1.  Click **Actions** from the unvolded menu.
1.  Click **New repository secret**
1.  Set the "Name" to `REPO_SSH_NAME` for reusability of our workflows
1.  At the **Deploy keys** step we have named the deploy key.  
    We need to set this as the "Secret"
1.  Head to the tab **Variables**
1.  Click **New repository variable**
1.  Set the "Name" to `REPO_NAME` for reusability of our workflows
1.  Set the "Value" to the repository name as it shows in the URL
1.  Head back the tab **Secrets**
1.  Now select **Manage organization secrets**  
    *(or let someone with acces do the following steps)*
1.  Add your repository for the following secrets:
    - `GHA_PW`
    - `GHA_SSH`
    - `GHA_USER`
    - `MAIN_IP`

## Docker

1.  Create the following dockerfiles for the project depending on environments  
    *(We do not include an example since they can be vastly different. Depending on project)*
    - `prd.dockerfile` 
    - `rcp.dockerfile` *(optional)*
    - `acc.dockerfile` *(optional)*
    - `dev.dockerfile`

1.  Setup a `docker-compose.yml`
    ```
    version: '3'

    services:
      prd:
        container_name: <repo-name>-prd
        build:
          context: .
          dockerfile: prd.dockerfile
          ports:
            -  '0000:0000'
      acc:
        container_name: <repo-name>-acc
        build:
          context: .
          dockerfile: acc.dockerfile
          ports:
            -  '0001:0001'
      dev:
        container_name: <repo-name>-dev
        build:
          context: .
          dockerfile: dev.dockerfile
          ports:
            -  '0002:0002'
    ```
    Keep the services that are relevant for the project or add more when missing
1.  Make sure locally `docker compose up <service>` works before continueing

## Nginx

To have the reverse proxy pickup our project we have `/nginx` folder in the root of the repository. Here we will have the following configs:
*(We do not include an example since they can be vastly different. Depending on project)*
- `prd.conf`
- `rcp.conf` *(optional)*
- `acc.conf` *(optional)*
- `dev.conf`

## Workflows

We currently have 2 default workflows copy them to the `/.github/workflows` directory or create it.

- Production `deploy-prd.yml`
    ```
    name: Deploy Node.js CI - DO-001

    on:
      push:
        branches: [main]

    jobs:
      build:
        runs-on: [ubuntu-22.04]
        env:
          NODE_ENV: production

        steps:
          - name: Deploy production
            uses: appleboy/ssh-action@master
            with:
              host: ${{ secrets.MAIN_IP }}
              username: ${{ secrets.GHA_USER }}
              key: ${{ secrets.GHA_SSH }}
              passphrase: ${{ secrets.GHA_PW }}
              script: |
                eval `ssh-agent -s`
                cd ./_server/${{ vars.REPO_NAME }}/prd/
                rm -rf ./${{ vars.REPO_NAME }}
                ssh-add ~/.ssh/${{ secrets.REPO_SSH_NAME }}
                git clone git@github.com:squeeble-ink/${{ vars.REPO_NAME }}.git
                cd ./${{ vars.REPO_NAME }}
                rm ./nginx/dev.conf
                docker compose up prd -d --build
    ```

- Development `deploy-dev.yml`
    ```
    name: Deploy Node.js CI - DO-001

    on:
      push:
        branches: [develop]

    jobs:
      build:
        runs-on: [ubuntu-22.04]

        steps:
          - name: Deploy development
            uses: appleboy/ssh-action@master
            with:
              host: ${{ secrets.MAIN_IP }}
              username: ${{ secrets.GHA_USER }}
              key: ${{ secrets.GHA_SSH }}
              passphrase: ${{ secrets.GHA_PW }}
              script: |
                eval `ssh-agent -s`
                cd ./_server/${{ vars.REPO_NAME }}/dev/
                rm -rf ./${{ vars.REPO_NAME }}
                ssh-add ~/.ssh/${{ secrets.REPO_SSH_NAME }}
                git clone git@github.com:squeeble-ink/${{ vars.REPO_NAME }}.git
                cd ./${{ vars.REPO_NAME }}
                git checkout develop
                rm ./nginx/prd.conf
                docker compose up dev -d --build
    ```

When a project needs additional environment variables please do not forget to update the following files:
- `/.github/workflows/deploy-*.yml`
- `/docker-compose.yml`
- `/*.dockerfile`

To fully deploy the repository we need to make sure these files get on the `develop` and `main` branch via Pull Requests

When that is all allright we need to do is a reload on the nginx instance on the server(s)
So ssh in there on the correct account
```
sudo nginx -t
```
Check if the new nginx configs from the project are correct and if so run the next command.
```
sudo systemctl restart nginx
```
If this project has its own domain do not forget to update the DNS servers to point to the correct one.
