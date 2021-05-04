# Kubernetes & OpenShift demo

In this demo, we'll be deploying a simple application to a Kubernetes cluster to get familiar with the structure of Kubernetes deployment templates as well as the CLI tools `oc` and `kubectl`.

## Part 0: Useful terminal setup
Using `oc` and `kubectl` from the commandline involves a whole lot of typing. It's really useful to have autocompletion for command names, arguments, resource types, and object names. To enable this (if it's not already in your `.bashrc`), you can run
```bash
source <(kc completion bash); source <(oc completion bash)
```
To get more information, use `oc completion --help`. (Note: if you're using `zsh`, that's supported too! Just replace `bash` with `zsh` above)

## Part 1: Deploying the application

The `templates/` directory contains the YAML files we'll be deploying to our cluster. 

You can *apply* a template to the cluster using the command
```bash
oc apply -f templates/<file>.yaml
```
You can also use
```bash
oc apply -f templates/
```
to apply *all* files in the templates folder at once.

____

First, lets create our deployment:
```bash
oc apply -f templates/deployment.yaml
```

we can check the status of our deployment using `oc get` and `oc describe`
```bash
oc get deployment my-deployment
# output:
NAME              READY   UP-TO-DATE   AVAILABLE   AGE
demo-deployment   1/3     3            1           2s
```
we can see that 1/3 replicas are ready. While we wait for everything to get started, we can also:
1. Check out the pods created by the deployment using `oc get po -l 'app=demo-app'`. Here, we're using a *label selector* to only get pods with label `app=demo-app`. Otherwise, we'd get all pods in the current namespace.
2. View the yaml we used for our deployment, with its current status using `oc get deploy demo-deployment -o yaml` (you can use `-o json` to output JSON as well!)

We've now got a deployment that contains our server running in Kubernetes! How do we access it from our browser?

First, we need to create a service to route traffic to the set of pods maintained by our deployment:
```bash
oc apply -f templates/service.yaml
```
and then we need to create a route to expose that service to the internet:
```bash
oc apply -f templates/route.yaml
```
Note: if you're running this demo on Kubernetes instead of OpenShift, routes will not be available. In this case, you'll need to use `templates/ingress.yaml`, but see the comments in that file for manual configuration you'll need to do!

## Testing it out
Once you've created the route, you can get the URL you'll use to access the deployment from `oc`:
```bash
oc get routes
# output
NAME         HOST/PORT                                          PATH   SERVICES       PORT   TERMINATION   WILDCARD
demo-route   demo-route-<current-namespace>.<url-for-cluster>          demo-service   http                 None
```
On Kubernetes, you'd look for the URL in the output of `kubectl get ingress`

Accessing this URL should show you a very plain HTML page with a link that doesn't lead to anything useful.

## Configuring our deployment
If you look at `deployment.yaml`, you'll see sections for `volumes` and `volumeMounts`. This is what we use to mount a configmap into the pod's containers. Note that the volume section of deployment.yaml specifies `optional: true`:
```yaml
# ...
volumes:
  - name: our-configmap
    configMap:
      name: demo-configmap
      optional: true
```
This allows the deployment to start even if the configmap doesn't exist (otherwise, the pod would wait for the configmap to be created before starting). 

We can now create our configmap using
```bash
oc apply -f templates/configmap.yaml
```
However, the deployment is already running and won't see our configmap immediately. To trigger a new deployment and pick up the configmap, we can use `oc rollout`:
```bash
oc rollout restart deployment "demo-deployment"
```
If you check what pods are running after doing this (`oc get pods`), you'll see a *rolling* deployment in action. This update strategy keeps your desired number of replicas running at all times -- it creates a new pod, waits for it to be running, and then terminates an old one. This way, your application can update without any downtime.

Once our rollout is finished, we can reload the URL for the application and see that the link on the landing page leads to a file defined in our configmap :)

## Other things to try (for fun?)
Here's a few other things you can play with to get familiar with OpenShift:
1. Try deleting a pod using `oc delete pod <pod-name>` and notice that the deployment automatically recreates it.
    - You can get a pod name from `oc get pods`
2. Scale the deployment up to five replicas using `oc scale deploy demo-deployment --replicas=5`
3. Try applying `templates/pod.yaml` to the cluster. What happens if you `oc delete` this pod?
4. Execute some commands inside a pod using `oc exec -it <pod-name> -- /bin/bash`. You can even edit `/usr/local/apache2/htdocs/index.html` to serve up a different landing page (but note these changes will be gone when the pod terminates!)
5. Begin getting familiar with the *short names* for Kubernetes resources. Most objects we've dealt with can be written as a short form in commands, e.g. `po` instead of `pods`, `svc` instead of `service`. You can see a list of resources your cluster understands by executing `oc api-resources` (be warned: the list is overwhelmingly long). Short names are listed in the second column
    - You can also have `oc` explain what a resource is to you using `oc explain <resource>`. This extends to fields in that resource, so you can, for example, use `oc explain pods.status` to get a description of all fields in a pod's status.

## Cleanup
To remove everything we've deployed thus far, you can use a label selector to delete all the objects we created in the demo:
```bash
oc delete all -l 'app=demo-app'
```