## GuestBook example

This example shows how to build a simple multi-tier web application using Kubernetes and Docker.

The example combines a web frontend, a redis master for storage and a replicated set of redis slaves.

### Step Zero: Prerequisites

This example assumes that you have a basic understanding of kubernetes services and that you have forked the repository and [turned up a Kubernetes cluster](https://github.com/GoogleCloudPlatform/kubernetes#contents):

```shell
$ cd kubernetes
$ hack/dev-build-and-up.sh
```

### Step One: Turn up the redis master

Use the file `examples/guestbook/redis-master-controller.json` which describes a single pod running a redis key-value server in a container.
Note that, although the redis server runs just with a single replica, we use replication controller to enforce that exactly one pod keeps running (e.g. in a event of node going down, the replication controller will ensure that the redis master gets restarted on a healthy node). 

```js
{
  "id": "redis-master-controller",
  "kind": "ReplicationController",
  "apiVersion": "v1beta1",
  "desiredState": {
    "replicas": 1,
    "replicaSelector": {"name": "redis-master"},
    "podTemplate": {
      "desiredState": {
        "manifest": {
          "version": "v1beta1",
          "id": "redis-master",
          "containers": [{
            "name": "redis-master",
            "image": "dockerfile/redis",
            "cpu": 100,
            "ports": [{"containerPort": 6379}]
          }]
        }
      },
      "labels": {
        "name": "redis-master",
        "app": "redis"
      }
    }
  },
  "labels": {
    "name": "redis-master"
  }
}
```

Create the redis pod in your Kubernetes cluster by running:

```shell
$ cluster/kubectl.sh create -f examples/guestbook/redis-master-controller.json

$ cluster/kubectl.sh get rc
CONTROLLER                             CONTAINER(S)            IMAGE(S)                                 SELECTOR                     REPLICAS
redis-master-controller                redis-master            dockerfile/redis                         name=redis-master            1
```

Once that's up you can list the pods in the cluster, to verify that the master is running:

```shell
$ cluster/kubectl.sh get pods
```

You'll see a single redis master pod. It will also display the machine that the pod is running on once it gets placed (may take up to thirty seconds):

```
POD                                          IP                  CONTAINER(S)            IMAGE(S)                                 HOST                                                           LABELS                                                     STATUS                                                                                                                               
redis-master-controller-gb50a                10.244.3.7          redis-master            dockerfile/redis                         kubernetes-minion-4.c.hazel-mote-834.internal/104.154.54.203   app=redis,name=redis-master                                Running

```

If you ssh to that machine, you can run `docker ps` to see the actual pod:

```shell
me@workstation$ gcloud compute ssh kubernetes-minion-4

me@kubernetes-minion-2:~$ sudo docker ps
CONTAINER ID        IMAGE                                  COMMAND                CREATED              STATUS              PORTS                    NAMES
0ffef9649265        dockerfile/redis:latest                "redis-server /etc/r   About a minute ago   Up About a minute                            k8s_redis-master.767aef46_redis-master-controller-gb50a.default.api_4530d7b3-ae5d-11e4-bf77-42010af0d719_579ee964                   
```

(Note that initial `docker pull` may take a few minutes, depending on network conditions.  During this time, the `get pods` command will return `Pending` because the container has not yet started )

### Step Two: Turn up the master service

A Kubernetes 'service' is a named load balancer that proxies traffic to one or more containers. The services in a Kubernetes cluster are discoverable inside other containers via *environment variables*. Services find the containers to load balance based on pod labels.  These environment variables are typically referenced in application code, shell scripts, or other places where one node needs to talk to another in a distributed system.  You should catch up on [kubernetes services](https://github.com/GoogleCloudPlatform/kubernetes/blob/master/docs/services.md) before proceeding.

The pod that you created in Step One has the label `name=redis-master`. The selector field of the service determines which pods will receive the traffic sent to the service. Use the file `examples/guestbook/redis-master-service.json`:

```js
{
  "id": "redis-master",
  "kind": "Service",
  "apiVersion": "v1beta1",
  "port": 6379,
  "containerPort": 6379,
  "selector": {
    "name": "redis-master"
  },
  "labels": {
    "name": "redis-master"
  }
}
```

to create the service by running:

```shell
$ cluster/kubectl.sh create -f examples/guestbook/redis-master-service.json
redis-master

$ cluster/kubectl.sh get services
NAME                    LABELS                                    SELECTOR                     IP                  PORT
redis-master            name=redis-master                         name=redis-master            10.0.246.242        6379
```

This will cause all pods to see the redis master apparently running on <ip>:6379.

Once created, the service proxy on each minion is configured to set up a proxy on the specified port (in this case port 6379).

### Step Three: Turn up the replicated slave pods
Although the redis master is a single pod, the redis read slaves are a 'replicated' pod. In Kubernetes, a replication controller is responsible for managing multiple instances of a replicated pod.

Use the file `examples/guestbook/redis-slave-controller.json`:

```js
{
  "id": "redis-slave-controller",
  "kind": "ReplicationController",
  "apiVersion": "v1beta1",
  "desiredState": {
    "replicas": 2,
    "replicaSelector": {"name": "redis-slave"},
    "podTemplate": {
      "desiredState": {
         "manifest": {
           "version": "v1beta1",
           "id": "redis-slave",
           "containers": [{
             "name": "redis-slave",
             "image": "brendanburns/redis-slave",
             "cpu": 200,
             "ports": [{"containerPort": 6379}]
           }]
         }
      },
      "labels": {
        "name": "redis-slave",
        "uses": "redis-master",
        "app": "redis"
      }
    }
  },
  "labels": {"name": "redis-slave"}
}
```

to create the replication controller by running:

```shell
$ cluster/kubectl.sh create -f examples/guestbook/redis-slave-controller.json
redis-slave-controller

$ cluster/kubectl.sh get rc
CONTROLLER                             CONTAINER(S)            IMAGE(S)                                 SELECTOR                     REPLICAS
redis-master-controller                redis-master            dockerfile/redis                         name=redis-master            1
redis-slave-controller                 redis-slave             brendanburns/redis-slave                 name=redis-slave             2
```

The redis slave configures itself by looking for the Kubernetes service environment variables in the container environment. In particular, the redis slave is started with the following command:

```shell
redis-server --slaveof ${REDIS_MASTER_SERVICE_HOST:-$SERVICE_HOST} $REDIS_MASTER_SERVICE_PORT
```

You might be curious about where the *REDIS_MASTER_SERVICE_HOST* is coming from.  It is provided to this container when it is launched via the kubernetes services, which create environment variables (there is a well defined syntax for how service names get transformed to environment variable names in the documentation linked above).

Once that's up you can list the pods in the cluster, to verify that the master and slaves are running:

```shell
$ cluster/kubectl.sh get pods
POD                                          IP                  CONTAINER(S)            IMAGE(S)                                 HOST                                                           LABELS                                                     STATUS
redis-master-controller-gb50a                10.244.3.7          redis-master            dockerfile/redis                         kubernetes-minion-4.c.hazel-mote-834.internal/104.154.54.203   app=redis,name=redis-master                                Running
redis-slave-controller-182tv                 10.244.3.6          redis-slave             brendanburns/redis-slave                 kubernetes-minion-4.c.hazel-mote-834.internal/104.154.54.203   app=redis,name=redis-slave,uses=redis-master               Running
redis-slave-controller-zwk1b                 10.244.2.8          redis-slave             brendanburns/redis-slave                 kubernetes-minion-3.c.hazel-mote-834.internal/104.154.54.6     app=redis,name=redis-slave,uses=redis-master               Running
```

You will see a single redis master pod and two redis slave pods.

### Step Four: Create the redis slave service

Just like the master, we want to have a service to proxy connections to the read slaves. In this case, in addition to discovery, the slave service provides transparent load balancing to clients. The service specification for the slaves is in `examples/guestbook/redis-slave-service.json`:

```js
{
  "id": "redisslave",
  "kind": "Service",
  "apiVersion": "v1beta1",
  "port": 6379,
  "containerPort": 6379,
  "labels": {
    "name": "redis-slave"
  },
  "selector": {
    "name": "redis-slave"
  }
}
```

This time the selector for the service is `name=redis-slave`, because that identifies the pods running redis slaves. It may also be helpful to set labels on your service itself as we've done here to make it easy to locate them with the `cluster/kubectl.sh get services -l "label=value"` command.

Now that you have created the service specification, create it in your cluster by running:

```shell
$ cluster/kubectl.sh create -f examples/guestbook/redis-slave-service.json
redisslave

$ cluster/kubectl.sh get services
NAME                    LABELS                                    SELECTOR                     IP                  PORT
redis-master            name=redis-master                         name=redis-master            10.0.246.242        6379
redisslave              name=redis-slave                          name=redis-slave             10.0.72.62          6379
```

### Step Five: Create the frontend pod

This is a simple PHP server that is configured to talk to either the slave or master services depending on whether the request is a read or a write. It exposes a simple AJAX interface, and serves an angular-based UX. Like the redis read slaves it is a replicated service instantiated by a replication controller.

The pod is described in the file `examples/guestbook/frontend-controller.json`:

```js
{
  "id": "frontend-controller",
  "kind": "ReplicationController",
  "apiVersion": "v1beta1",
  "desiredState": {
    "replicas": 3,
    "replicaSelector": {"name": "frontend"},
    "podTemplate": {
      "desiredState": {
         "manifest": {
           "version": "v1beta1",
           "id": "frontend",
           "containers": [{
             "name": "php-redis",
             "image": "kubernetes/example-guestbook-php-redis",
             "cpu": 100,
             "memory": 50000000,
             "ports": [{"name": "http-server", "containerPort": 80}]
           }]
         }
       },
       "labels": {
         "name": "frontend",
         "uses": "redis-slave,redis-master",
         "app": "frontend"
       }
      }},
  "labels": {"name": "frontend"}
}
```

Using this file, you can turn up your frontend with:

```shell
$ cluster/kubectl.sh create -f examples/guestbook/frontend-controller.json
frontend-controller

$ cluster/kubectl.sh get rc
CONTROLLER                             CONTAINER(S)            IMAGE(S)                                 SELECTOR                     REPLICAS
frontend-controller                    php-redis               kubernetes/example-guestbook-php-redis   name=frontend                3
redis-master-controller                redis-master            dockerfile/redis                         name=redis-master            1
redis-slave-controller                 redis-slave             brendanburns/redis-slave                 name=redis-slave             2
```

Once that's up (it may take ten to thirty seconds to create the pods) you can list the pods in the cluster, to verify that the master, slaves and frontends are running:

```shell
$ cluster/kubectl.sh get pods
POD                                          IP                  CONTAINER(S)            IMAGE(S)                                 HOST                                                           LABELS                                                     STATUS
frontend-controller-5m1zc                    10.244.1.131        php-redis               kubernetes/example-guestbook-php-redis   kubernetes-minion-2.c.hazel-mote-834.internal/146.148.71.71    app=frontend,name=frontend,uses=redis-slave,redis-master   Running
frontend-controller-ckn42                    10.244.2.134        php-redis               kubernetes/example-guestbook-php-redis   kubernetes-minion-3.c.hazel-mote-834.internal/104.154.54.6     app=frontend,name=frontend,uses=redis-slave,redis-master   Running
frontend-controller-v5drx                    10.244.0.128        php-redis               kubernetes/example-guestbook-php-redis   kubernetes-minion-1.c.hazel-mote-834.internal/23.236.61.63     app=frontend,name=frontend,uses=redis-slave,redis-master   Running
redis-master-controller-gb50a                10.244.3.7          redis-master            dockerfile/redis                         kubernetes-minion-4.c.hazel-mote-834.internal/104.154.54.203   app=redis,name=redis-master                                Running
redis-slave-controller-182tv                 10.244.3.6          redis-slave             brendanburns/redis-slave                 kubernetes-minion-4.c.hazel-mote-834.internal/104.154.54.203   app=redis,name=redis-slave,uses=redis-master               Running
redis-slave-controller-zwk1b                 10.244.2.8          redis-slave             brendanburns/redis-slave                 kubernetes-minion-3.c.hazel-mote-834.internal/104.154.54.6     app=redis,name=redis-slave,uses=redis-master               Running
```

You will see a single redis master pod, two redis slaves, and three frontend pods.

The code for the PHP service looks like this:

```php
<?

set_include_path('.:/usr/share/php:/usr/share/pear:/vendor/predis');

error_reporting(E_ALL);
ini_set('display_errors', 1);

require 'predis/autoload.php';

if (isset($_GET['cmd']) === true) {
  header('Content-Type: application/json');
  if ($_GET['cmd'] == 'set') {
    $client = new Predis\Client([
      'scheme' => 'tcp',
      'host'   => getenv('REDIS_MASTER_SERVICE_HOST') ?: getenv('SERVICE_HOST'),
      'port'   => getenv('REDIS_MASTER_SERVICE_PORT'),
    ]);
    $client->set($_GET['key'], $_GET['value']);
    print('{"message": "Updated"}');
  } else {
    $read_port = getenv('REDIS_MASTER_SERVICE_PORT');

    if (isset($_ENV['REDISSLAVE_SERVICE_PORT'])) {
      $read_port = getenv('REDISSLAVE_SERVICE_PORT');
    }
    $client = new Predis\Client([
      'scheme' => 'tcp',
      'host'   => getenv('REDIS_MASTER_SERVICE_HOST') ?: getenv('SERVICE_HOST'),
      'port'   => $read_port,
    ]);

    $value = $client->get($_GET['key']);
    print('{"data": "' . $value . '"}');
  }
} else {
  phpinfo();
} ?>
```

### Step Six: Create the guestbook service.

Just like the others, you want a service to group your frontend pods.

The service is described in the file `examples/guestbook/frontend-service.json`:

```js
{
  "id": "frontend",
  "kind": "Service",
  "apiVersion": "v1beta1",
  "port": 8000,
  "containerPort": "http-server",
  "selector": {
    "name": "frontend"
  },
  "labels": {
    "name": "frontend"
  },
  "createExternalLoadBalancer": true
}
```

```shell
$ cluster/kubectl.sh create -f examples/guestbook/frontend-service.json
frontend

$ cluster/kubectl.sh get services
NAME                    LABELS                                    SELECTOR                     IP                  PORT
frontend                name=frontend                             name=frontend                10.0.93.211         8000
redis-master            name=redis-master                         name=redis-master            10.0.246.242        6379
redisslave              name=redis-slave                          name=redis-slave             10.0.72.62          6379
```

You may need to open the firewall for port 80 using the [console][cloud-console] or the `gcloud` tool. The following command will allow traffic from any source to instances tagged `kubernetes-minion`:

```shell
$ gcloud compute firewall-rules create --allow=tcp:8000 --target-tags=kubernetes-minion kubernetes-minion-8000
```

`cluster/kubectl.sh` automatically creates forwarding rule for services with `createExternalLoadBalancer`.

```shell
$ gcloud compute forwarding-rules list
NAME                  REGION      IP_ADDRESS     IP_PROTOCOL TARGET
frontend              us-central1 130.211.188.51 TCP         us-central1/targetPools/frontend
```

Finally, you can access the guestbook through `IP_ADDRESS 130.211.188.51` associated with forwarding rule on port 8000.
For details about limiting traffic to specific sources, see the [GCE firewall documentation][gce-firewall-docs].

[cloud-console]: https://console.developer.google.com
[gce-firewall-docs]: https://cloud.google.com/compute/docs/networking#firewalls

### Step Six: Cleanup

To turn down a Kubernetes cluster:

```shell
$ cluster/kube-down.sh
```
