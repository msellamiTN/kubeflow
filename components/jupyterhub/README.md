# Jupyter and Jupyterhub

## Background

### Jupyter
Jupyter (formerly iPython Notebook) is a UI tool commonly used with Spark, Tensorflow and other big data processing frameworks. It is used
by data scientists and ML engineers across a variety of organizations for interactive tasks. It supports multiple languages through runners,
and allows users to run code, save code/results, and share “notebooks” with both code, documentation and output easily. 

### JupyterHub
JupyterHub lets users manage authenticated access to multiple single-user Jupyter notebooks. JupyterHub delegates the launching of
single-user notebooks to pluggable components called “spawners”. JupyterHub has a sub-project named kubespawner, maintained by the
community, that enables users to provision single-user Jupyter notebooks backed by Kubernetes pods - the notebooks themselves are
Kubernetes pods.

## Quick Start

Refer to the [user_guide](../../user_guide.md) for instructions on deploying JupyterHub via ksonnet.

Once that's completed, you will have a StatefulSet for Jupyterhub, a configmap for configuration, and a LoadBalancer type of service, in addition to the requisite RBAC roles.
If you are on Google Kubernetes Engine, the LoadBalancer type of service automatically creates an external IP address that can be 
used to access the notebook. Note that this is for illustration purposes only, and must be coupled with [SSL](http://jupyterhub.readthedocs.io/en/0.8.1/getting-started/security-basics.html?highlight=ssl#ssl-encryption) and configured to use an
[authentication plugin](https://github.com/willingc/jhubdocs/blob/master/jupyterhub/docs/source/authenticators.md) in production environments.

if you're testing and want to avoid exposing JupyterHub on an external IP address, you can use kubectl instead to gain access to the hub on your local machine.

```commandline
kubectl port-forward <jupyterhub-pod-name> 8000:8000
``` 

The above will expose JupyterHub on http://localhost:8000. The pod name can be obtained by running `kubectl get pods`, and will be `tf-hub-0` by default. 

## Configuration

Configuration for JupyterHub is shipped separately and contained within the configmap defined by the [core componenent](https://github.com/google/kubeflow/tree/master/kubeflow). It is a Python file that is consumed by JupyterHub on starting up. The supplied configuration has reasonable defaults for the requisite fields and **no authenticator** configured by default. Furthermore, we provide a number of parameters that can be used to configure
the core component. To see a list of ksonnet parameters run

```
ks prototype describe kubeflow-core
```

If the provided parameters don't provide the flexibility you need, you can take advantage of ksonnet to customize the core component and use a config file fully specified by you.

Configuration includes sections for KubeSpawner and Authenticators. Spawner parameters include the form used when provisioning new 
Jupyter notebooks, and configuration defining how JupyterHub creates and interacts with Kubernetes pods for individual notebooks. 
Authenticator parameters correspond to the authentication mechanism used by JupyterHub. 

 
## Usage

If you're using the quick-start, the external IP address of the JupyterHub instance can be obtained from `kubectl get svc`.
```commandline
 kubectl get svc
 
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)        AGE
tf-hub-0       ClusterIP      None            <none>          <none>         1h
tf-hub-lb    LoadBalancer   10.43.246.148   xx.yy.zz.ww   80:32689/TCP   36m
``` 

Now, you can access http://xx.yy.zz.ww with your browser. When trying to spawn a new image, a configuration page should pop up, allowing configuration of the notebook image, CPU, Memory, and additional resources. With the default `DummyAuthenticator`, it should allow any username/password to access the hub and create new notebooks. You can use an authenticator plugin if you want to secure your notebook server and use its administration functionality. 

## Customization

### Using your own hub image

An image with JupyterHub 0.8.1, kubespawner 0.7.1 and two simple authenticator plugins can be built from within the `docker/` directory using the Makefile provided. For example, if you're using Google Cloud Platform and have a project with ID `foo` configured to use gcr.io, you can do the following: 

```commandline
make build PROJECT_ID=foo
make push PROJECT_ID=foo
```

### Notebook image

Images published under https://github.com/jupyter/docker-stacks should work directly with the Hub. The only requirement for the jupyter
notebook images that can be used in conjunction with this instance of Hub is that the same version of JupyterHub must be installed (0.8.1 by default), and that there must be a standard `start-singleuser.sh` accessible via the default PATH. 

### GitHub OAuth Setup

After creating the initial Hub and exposing it on a public IP address, you can add GitHub based authentication. First, you'll need to create a [GitHub oauth application](https://github.com/settings/applications/new). The callback URL would be of the form `http://xx.yy.zz.ww/hub/oauth_callback`. 

Once the GitHub application is created, update the `manifest/config.yaml` with the `callback_url`, `client_id` and `client_secret` obtained from the GitHub UI. Ensure that the `DummyAuthenticator` is commented out and replaced by the `GitHubOAuthenticator` options. At the end, the authenticator configuration section might look like:

```commandline
c.JupyterHub.authenticator_class = GitHubOAuthenticator
c.GitHubOAuthenticator.oauth_callback_url = 'http://xx.yy.zz.ww/hub/oauth_callback'
c.GitHubOAuthenticator.client_id = 'client_id_here'
c.GitHubOAuthenticator.client_secret = 'client_secret_here'
```

Finally, you can update the configuration and ensure that the new configuration is picked up, by doing the following:

```commandline
ks apply ${ENVIRONMENT} -c ${COMPONENT_NAME}
kubectl delete pod tf-hub-0
```

The pod will come up with the new configuration and be configured to use the GitHub authenticator you specified in the previous step. You can additionally modify the configuration to add whitelists and admin users. For example, to limit it to only GitHub users user1 and user2, one might use the following configuration:

```
c.Authenticator.whitelist = {'user1', 'user2'}
```

After changing the configuration and `kubectl apply -f config.yaml`, please note that the JupyterHub pod needs to be restarted before  the new configuration is reflected.