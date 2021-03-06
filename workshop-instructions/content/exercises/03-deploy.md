This exercise will walk you through how to deploy a containerized web app to
a modern *container orchestration* platform.

The platform you will deploy to is
[*Kubernetes*](https://kubernetes.io).

# Learning Outcomes

After completing this lesson, you will be able to:

-   Describe the resources required to deploy a containerized web app
    to a container orchestration platform.

-   Deploy a blocking web application to a modern container
    orchestration platform.

# Getting started

Review the following slides and/or associated lectures:

-   Accessing your platform - Lecture #7 ([Slides](https://docs.google.com/presentation/d/1ctnmoRcgbcVyBndFBDr00HtE23ky8GSqFIgKkRmo2Tw/present))

-   Deploying an app - Lecture #8 ([Slides](https://docs.google.com/presentation/d/1mzZurB3sDo-7_Rj0p6RKv_gajg1rvFitPUd02Yvr-f8/present))

You are provided access to a Kubernetes cluster as your container
orchestration platform to which you will deploy your application.

# Review the runtime deployment configuration

1.  Make sure you are in your `~/exercises/k8s` directory now in
    both of your terminal windows,
    and clear both:

    ```terminal:execute-all
    command: cd ~/exercises/k8s && clear
    ```

1.  Review the list of files in the `~/exercises/k8s` directory.

    ```terminal:execute
    command: ls -al
    session: 1
    ```

1.  Notice the three yaml files.
    They comprise the *Runtime Configuration* of your application.

1.  Also notice these files are not stored with your application code.

    The *intent* of the [configuration in the environment](https://12factor.net/config)
    guideline is that you segregrate the code from the
    environment specific information,
    as they will vary along different dimensions (features vs runtime)
    and at different rates (one codebase, many deployments).

1.  Each of the yaml files comprise specific functions necessary to run
    and access your web application using the
    [Kubernetes](https://kubernetes.io) container orchestration platform.

    You will become familiar with each Kubernetes *resource* necessary
    to deploy your app in the remainder of this exercise.

# Access your platform

1.  It is necessary to access the platform via a self-service API.
    You do not have direct access to the machine(s) where your
    application will run.

1.  You have the `kubectl` command installed and pre-configured to
    authenticate and connect to a Kubernetes cluster.

1.  Review the `KUBECONFIG` file in your environment,
    this is a similar pattern that platforms use to provide
    authentication and context to a specific platform installation:

    ```terminal:execute
    command: kubectl config view
    session: 1
    ```

# Platform resources and isolation

1.  Platform operators have set up rules to provision a fixed amount of
    resources available to you to run the lesson,
    as well as a dedicated/isolated space that other app operators on
    the platform will not see.

1.  The Kubernetes feature that provides this functionality is the
    [namespace](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)

1.  You cannot access other user namespaces.

# Verify access and namespace

1.  To verify that `kubectl` is correctly configured, run the following

    ```terminal:execute
    command: kubectl get pod,svc,deploy
    session: 1
    ```

1.  This command returns a list of pods, services, and deployments in your dedicated *namespace*.
    You should see an output similar to the following:

    ```no-highlight
    No resources found in {{ session_namespace }} namespace.
    ```

-   Since you have not yet deployed your application,
    you will see the message prefix of
    `No resources found`.

-   Notice the `{{ session_namespace }}` is the *namespace*
    of this example -
    you will likely see a different value as your dedicated *namespace*
    for this workshop.

# Deploy your application

Modern Platforms (including PaaS and CaaS) have a concept of
*Application* or *Deployment* that is a high level abstraction over a
set of features comprising a product.
But the *Application* or *Deployment* may have many runtime components
(such as containers) comprising the application.

Kubernetes has first class support for *Deployment* resource to help you
deploy your application workloads.

1.  Review the `~/exercises/k8s/deployment.yaml` file:

    ```editor:open-file
    file: ~/exercises/k8s/deployment.yaml
    ```

    There is a lot going on here,
    but if you focus on the `name` attributes,
    you can figure out what is going on:

    -   `pal-tracker`:
        name of the Kubernetes *Deployment* resource.

    -   `pal-tracker-pod`:
        Name of the Kubernetes *Pod* inside of the
        *Deployment*.

        *Pods* are units of isolation that can group multiple containers
        to accomplish a unit of work.
        They can also be "scaled" by adding more of them in a deployment.

    -   `pal-tracker-container`
        Name of the container running as part
        of a *Pod*, as part of a *Deployment*.

        You can see it reference to the container image you built
        and published in the last exercise.

    Notice that there is no specification for routing in the *Deployment*
    resource.
    In Kubernetes,
    the *Deployment* resource has the sole responsibility of maintaining
    the running number of workers in your application.
    You will see routing accomplished with the other two resources later
    in this lesson.

1.  Apply your Kubernetes Deployment by running:

    ```terminal:execute
    command: kubectl apply -f deployment.yaml
    session: 1
    ```

1.  Watch your Deployment as the Kubernetes objects are created by
    running the following:

    ```terminal:execute
    command: kubectl rollout status deployment/pal-tracker --watch
    session: 1
    ```

    This will allow you to see when your `Deployment` is successfully
    deployed.

1.  To see a snapshot of the Kubernetes objects for your
    Deployment, run:

    ```terminal:execute
    command: kubectl get deployments
    session: 2
    ```

    You should see `pal-tracker` listed as a Deployment.
    Since this is a snapshot you may see that there is a 0 listed under
    the AVAILABLE column, if
    this is the case run `kubectl get deployments` again and wait for
    the Deployment to finish creating and you will see a 1 listed for
    AVAILABLE.

# Verify your Deployment

When applying the Kubernetes Deployment, behind the scenes it creates
a Kubernetes *Pod* object.
A *Pod* is a group of containers that are deployed together to
accomplish a specific work task in context of a deployment,
and can be scaled out to multiple pods if needed to accommodate more
load.

In your case, you are deploying a single container application.

1.  Check on the status of the Pod that was created by running:

    ```terminal:execute
    command: kubectl get pods
    session: 1
    ```

    Under the STATUS column you should see "Running".

1.  Take the Pod name from the previous command, and use it to run:

    ```workshop:copy-and-edit
    text: kubectl describe pod POD-NAME
    session: 1
    ```

    In the output of the above command, you should be able to verify
    that the Pod is running a container using the `pal-tracker` image you
    published to the container registry.
    If you need to debug any issues,
    take a look at the `Events` section of the `kubectl describe pod`
    output.

# Route traffic to your application

There are two problems to solve when routing an external inbound request
to a web application running on a container orchestration platform:

1.  How a web request from outside the platform makes it to inside the
    platform's container network.
    This is commonly called *Ingress access*.

2.  How to figure out to which container to route the traffic inside the
    platform.
    This is commonly called *Load Balancing*.

With Kubernetes,
there are two platform *resources* to help you out.

## Create and verify Kubernetes service

In the output of the `kubectl describe pod`, you might notice that
there is an IP address for the *Pod*.
Unfortunately this IP address is only accessible from within the
Kubernetes cluster.

Your application runs on port 8080 inside the container and its
associated Pod,
which in a hardened environment will not be accessible to
inside or outside the Kubernetes cluster without additional
work.

To address this issue,
as *Service* resource description is provided to you.

1.  Review the `~/exercises/k8s/service.yaml` file:

    ```editor:open-file
    file: ~/exercises/k8s/service.yaml
    ```

    This is a bit going on here,
    but important:

    -   Notice the name of the service is `pal-tracker`,
        and the referenced ports are on `8080`.
        This is an internal name that can be referenced
        like a hostname in DNS
        (Kubernetes in fact uses an internal DNS for name resolution),
        that could be referenced similarly like this within the
        container network:

        `http://pal-tracker:8080`

    -   Notice the `spec` section.
        The `selector` section points to the pods that you deployed
        previously in the exercise.

        Kubernetes will associated any pods with the same name and
        internal port of `8080` with the service name,
        and register the container ip and port combinations with the
        service name.

1.  Create the Service by applying the change:

    ```terminal:execute
    command: kubectl apply -f service.yaml
    session: 1
    ```

1.  Verify that the service was created by running:

    ```terminal:execute
    command: kubectl get services
    session: 1
    ```

    In the output you will see the Service you applied.
    You will notice the Service is of TYPE `ClusterIP` and has a
    CLUSTER-IP set.

    **Just like the Pod, this Cluster IP address is only accessible**
    **from within the cluster.**

    The next section will solve that problem.

1.  Verify the endpoints, the IP addresses of the pods, by describing
    the `pal-tracker` service:

    ```terminal:execute
    command: kubectl describe service pal-tracker
    session: 1
    ```

## Create and verify Kubernetes Ingress

Kubernetes provides multiple solutions for handling the problem of
external access to the platform network.

You will use a Kubernetes *Ingress* that uses an `Nginx` to route
the traffic.

Currently you only have one application (deployment) running on the cluster,
so you can route all traffic to the same place.

1.  Review the `~/exercises/k8s/ingress.yaml` file:

    ```editor:open-file
    file: ~/exercises/k8s/ingress.yaml
    ```

    This is a simple application with only one route and associated
    endpoint,
    so the resource description is simple:

    -   There is an ingress route already configured for you at:

        `pal-tracker.{{ session_namespace }}.{{ ingress_domain }}`

        It consists of two parts:

        -   Hostname:
            `pal-tracker.{{ session_namespace }}`

            where the suffix `{{ session_namespace }}` is your workshop
            session namespace assigned to you.
            You will see it referenced later in the instructions as
            `${SESSION_NAMESPACE}`

        -   Domain:
            `{{ ingress_domain }}`

            You will see it referenced later in the instructions as
            `${INGRESS_DOMAIN}`

        The platform operator configured the `Domain` for the cluster,
        and this workshop configured the `Hostname` for you.

        In your work scenarios,
        your PaaS platform may help automate you generating the route,
        or if it does not, you may have to work with your operators to
        allocate domain and/or host names for you.

    -   The backend `service` description points the ingress
        router to the `pal-tracker` service exposed for port `8080`
        for the default path `/`.

1.  Create the Ingress resource by applying the change using `kubectl`.

    ```terminal:execute
    command: kubectl apply -f ingress.yaml
    session: 1
    ```

1.  Verify the creation was successful by running `kubectl get ingress`.

    ```terminal:execute
    command: kubectl get ingress
    session: 1
    ```

1.  View details of the Ingress resource:

    ```terminal:execute
    command: kubectl describe ingress/pal-tracker
    session: 1

### Verify access to your application

At this point you have all the parts deployed to access your application.
Access the default backend via both the K8s cluster IP address,
as well as the default domain.

1.  Visit the domain of your application using your web browser:

    ```dashboard:open-url
    url: http://pal-tracker.{{ session_namespace }}.{{ ingress_domain }}
    ```

1.  You should see one of two scenarios:

    -   You may see a `502` or a `503` error code if you immediately
        applied the Ingress object,
        and immediately attempted to visit the application in your
        browser.

        Try waiting 5-10 seconds,
        and refresh your browser.

        This happens because your platform has to do the work to
        register your backend service with your ingress (web server)
        and that takes time.

        It also highlights that you need to consider occasional failures
        when you change the deployment of your application.

        You will see more discussion how to handle in a later workshop
        in this series.

    -   The `hello` message rendered by your pal-tracker application.

## Monitor your platform

[Octant](https://octant.dev/) gives us a web interface to help inspect
our Kubernetes cluster.
It is not the full story on monitoring,
for this lesson,
sufficient to start.

Use __Octant__ to view your Kubernetes cluster.

1.  Navigate to the Console.
    You should see Octant UI.

    ```dashboard:open-dashboard
    name: Console
    ```

1.  Click on the `Namespace Overview` icon in the left vertibal bar.
    (It is the second icon from the top.)
    Here you see all of the objects you have created in the selected
    namespace.
    You can drill into specific objects from this page.

1.  Under `Deployments`, click on the `pal-tracker` deployment.
    You will see the `Summary` view for this object.

1.  Select the `Resource Viewer` tab at the top of the page.
    This view shows the object graph associated with the currently
    selected object, the deployment.
    They should all be green.
    If there is a problem with your cluster, this is a good place to
    start troubleshooting.

# Run a smoke test

Run a smoke test against your deployment via the following command:

```terminal:execute
command: curl -i http://pal-tracker.${SESSION_NAMESPACE}.${INGRESS_DOMAIN}
session: 2
```

You should see 'hello' message in the output.

# Wrap

Notice that Kubernetes container orchestration demonstrates the concepts
of the system statement management reconciliation loop you saw in the
early lectures:

1.  The application operator designates the *desired state* by applying
    the Kubernetes deployment,
    service and ingress resource specfications through the
    `kubectl apply` command.

1.  Kubernetes orchestrator will take all the necessary steps to make
    the *active state* match the desired state:

    -   Schedule and reserve resources from the Kubernetes cluster upon
        which to run the resources necessary for the app.
    -   Pull images needed to run the containers and associated *Pods*
        and *Deployment*.
    -   Start the Pods (containers).
    -   Make the *Deployment* ready to do work.
    -   Provide the Routing of external traffic to the deployment
        through *Ingress* and *Service* resources.

Notice the complexity of all the moving parts.

Also notice there is a lot of complexity in defining the Kubernetes
resources necessary to deploy such a simple application.

In more complex scenarios,
the application operator has their work cut out for them.

If you are interested in using Kubernetes as the backbone of your
application deployments,
make sure to checkout the
[Kube Academy](https://kube.academy/) series of courses to get you
started or the following Tanzu Developer Center topics:

- [Container basics](https://tanzu.vmware.com/developer/workshops/lab-container-basics/)
- [Kubernetes fundamentals](https://tanzu.vmware.com/developer/workshops/lab-k8s-fundamentals/)
- [Spring on Kubernetes](https://tanzu.vmware.com/developer/workshops/spring-on-kubernetes/)

More advanced tools are provided to help on the Application Operator
journey for deploying Kubernetes hosted apps:

- [Carvel](https://carvel.dev/)
- [Getting started with Carvel](https://tanzu.vmware.com/developer/workshops/lab-getting-started-with-carvel/)
- [Helm](https://helm.sh/)

# Resources

- [Namespace](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)
- [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Pod](https://kubernetes.io/docs/concepts/workloads/pods/)
- [Service](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
