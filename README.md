## Prerequisites:
[Authentik installed](https://github.com/vptimmy/k8-authentik).  
[kubed](https://appscode.com/products/kubed/)

## Add the helm repo
`helm repo add harbor https://helm.goharbor.io`

## Deploy
`skaffold deploy`

## Post Install

### Change admin password
1) Visit https://registry.home.lab -> login in as admin / admin.
2) Change your default local admin password -> upper right -> admin -> change password

### Authentik, create OATH2 provider
3) Visit https://authentik.home.lab to set up an SSO application.  Follow the instructions at https://goauthentik.io/integrations/services/harbor/ to setup the Oauth2 provider.  Be sure to also include the [username scope](https://github.com/vptimmy/k8-authentik/blob/main/README.md)!!! Once that is done, create an application called Harbor and link the provider you created to it.

### Harbor, Setup OIDC
4) Visit https://registry.home.lab, login in locally as admin, and click on Configuration. Choose OIDC and provide the information from step #3.  For the username claim use "username".  If you do not see OIDC as an option, it is because you have created manual users.  You must delete all your manual created users in order to setup OIDC on Harbor.

## Service Account
We will create a service account in Authentik to communicating with Harbor.

Login into https://authentik.home.lab, go to the admin dashboard -> Directory -> Users and "Create Service Account".  I used the username "harbor-sa" as the username. Note the password and change it if you want.

Now login into https://registry.home.lab with the harbor-sa account.  Click on harbor-sa in the upper right hand corner, then choose "User Profile".  The "CLI Secret" is going to be used for `docker login registry.home.lab` and also in Kubernetes registry secret you create below.

We need a registry secret to hold a username and password so our deployments know how to communicate with Harbor.  I wanted this secret to be duplicated to all namespaces, so I used `kubed` to accomplish this.  If you don't like kubed, you can duplicate the secret to the namespaces manually.

```
kubectl create -n harbor-system secret docker-registry registry-harbor-sa \
--docker-username=harbor-sa \
--docker-password=theOIDCPasswordAbove \
--docker-server=registry.home.lab 
```

Now sync that secret to all namespaces by adding a kubed annotation
`kubectl annotate -n harbor-system secrets registry-harbor-sa kubed.appscode.com/sync="true"`

## Login to repo
Use the credentials from above
`docker login registry.home.lab`

## Proxy DockerHub so you store images locally
1) Login to https://registry.home.lab as a local db admin.
2) Go to Registries
3) Click "NEW ENDPOINT"
| Provider      | Docker Hub                 |
| Name          | dockerhub-proxy            | 
| Access ID     | Your docker ID - not email |
| Access Secret | Your docker password       |  
4) Now go to Projects -> New Project 
| Project Name  | dockerhub-proxy            |
| Proxy Cache   | on / true                  |
| Endpoint      | dockerhub-proxy            |

Now do a docker pull.  A copy of that image appears in your registry!
`docker docker pull registry.home.lab/dockerhub-proxy/library/python`


## Uninstall
`skaffold delete`
