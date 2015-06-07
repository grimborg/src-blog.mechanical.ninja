+++
date = "2015-06-07T13:41:13+02:00"
draft = false
title = "Deploying a private and secure Docker registry"
menu = "docker"
+++

Let's deploy a private Docker registry that we can use for solo development or in a small team.

To keep the setup simple we'll use Basic HTTP Auth instead of the more elaborate [token authentication](http://docs.docker.com/registry/spec/auth/token/).

We'll need:

- A server (I used the smallest DigitalOcean instance).
- A domain name.
- A SSL certificate for our domain, which can be purchased cheaply from e.g. SSLs.com. (Hopefully [Let's Encrypt](https://letsencrypt.org/) will make this [easier and cheaper](https://twitter.com/zoobab/status/607563008022355969) soon!)

## Setting up the server

To run the Docker registry we'll use the [`registry:2.0` container](https://docs.docker.com/registry/).

To handle SSL and authentication, we'll use nginx, specifically the [`jwilder/nginx-proxy` container](https://github.com/jwilder/nginx-proxy). This proxy will also make it easy for us to deploy other outward-facing containers.

On the server, any Linux distribution that runs a recent version of Docker will do. I used the latest version of CoreOS.

### Starting the Docker Registry

We'll keep the registry data out of the container, so that the data is persisted independently of the container's lifecycle.

Create a `$HOME/registry/storage` directory, and run the following to start the registry:

```
$ docker run -d \
    -p 127.0.0.1::5000
    -v $HOME/registry/storage:/registry \
    -e REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY=/registry \
    -e VIRTUAL_HOST=<your-domain-name> \
    --name registry \
    registry:2.0
```

The `VIRTUAL_HOST` setting will be used by `nginx-proxy` (see below).

To verify that it's running, first let's find out to which port the container's port 5000 is mapped. Run `docker ps` and look for a line containing something like `127.0.0.1:32774->5000/tcp`. In this case, the port is 32774.

Run `curl http://localhost:32774/v2/` (where 32774 is the port you saw on `docker ps`). The Docker registry will return `{}`.

### Obtaining a SSL certificate

If you already have a SSL certificate for your domain, skip this step.

Otherwise, you'll need to generate a certificate request. An easy way to do so is to head over to [DigiCert's OpenSSL CSR Wizard](https://www.digicert.com/easy-csr/openssl.htm) and fill in the form.

Click 'Generate' and paste the `openssl req...` command from the wizard onto your shell. You'll get a `<domain>.csr` file that you'll have to send to your SSL provider to obtain the certificate. I bought mine off SSLs.com for $3.88 (only 1 year). You'll also get a `<domain>.key`. We'll use both files later.

If you used a cheap provider you'll receive a bunch of certificates, which you'll need to concatenate into a bundle. In the case of a Comodo certificate, you'd have to do so as follows (the order matters!): 

```
$ cat <yourdomain>.crt \
    COMODORSADomainValidationSecureServerCA.crt \
    COMODORSAAddTrustCA.crt \
    AddTrustExternalCARoot.crt \
    > bundle.csr
```

### Setting up nginx-proxy


Create the directories to store the certificate, the passwords and the vhost settings:

```
mkdir -p $HOME/nginx/{certs,vhost.d,passwords}
```

#### SSL Certificate


Copy your certificate (or bundle) as `nginx/certs/<your-domain-name>.crt` and your `.key` file as `nginx/certs/<your-domain-name>.key`.

#### VHost Settings

Create a file under `nginx/vhost.d` named `<your-domain-name>`, with this contents:

```
client_max_body_size 100m;
auth_basic "Restricted";
auth_basic_user_file /etc/nginx/passwords/<your-domain-name>;
add_header Docker-Distribution-API-Version registry/2.0 always;
```

Tweak the `client_max_body_size` as desired. That will be the maximum upload size for Docker.

#### Password

For each user, run the following:

```
printf "<username>:$(openssl passwd -crypt <password>)n" \
>> $HOME/nginx/passwords/<your-domain-name>
```

This will result on a file named `nginx/passwords/<you-domain-name>` containing, for each user:

```
<username>:<password-hash>
```

#### Start the nginx proxy

Run the following:

```
$ docker run -d -p 443:443 \
    -v $HOME/nginx/certs:/etc/nginx/certs:ro \
    -v $HOME/nginx/vhost.d:/etc/nginx/vhost.d:ro \
    -v $HOME/nginx/passwords:/etc/nginx/passwords:ro \
    -v /var/run/docker.sock:/tmp/docker.sock:ro \
    --name nginx \
    jwilder/nginx-proxy
```

Optionally, you may want to see the nginx logs, which are discarded by default. To do this, create a `nginx/log` directory and add the following line to the previous command:

```
-v $HOME/nginx/log:/var/log/nginx
```

Running this `docker run` command will start `nginx` listening on port 443, and it will automatically set up a virtual host for our docker registry based on its `VIRTUAL_HOST` environment variable.

## Log in

To log into our Docker registry, simply run `docker login <your-domain-name>` and enter your username, password and e-mail address.

## Pushing a simple container

Now you can try pushing a simple container to your registry:

```
$ docker pull hello-world
$ docker tag hello-world <your-domain-name>/<your-username>/hello-world
$ docker push <your-domain-name>/<your-username>/hello-world
```
