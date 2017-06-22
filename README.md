# Docker experiments

Docker image: dalpo/friendlyhello

## Services

A `docker-compose.yml` file is a YAML file that defines how Docker containers should behave in production.

### Run your new load-balanced app

```
± % docker swarm init

Swarm initialized: current node (jhtotk3jtyx47yf6q1q18icq5) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-1ni13jix7l2ieer37izqjhligjj7g4b819ppwseiyx4vmwiczm-50zr7l6mqk5lqrwryam4r0ers \
    192.168.1.3:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

Now let’s run it. You have to give your app a name – here it is set to getstartedlab :

```
± % docker stack deploy -c docker-compose.yml getstartedlab
Creating network getstartedlab_webnet
Creating service getstartedlab_web
```

See a list of the five containers you just launched:

```
± % docker stack ps getstartedlab
ID            NAME                 IMAGE                       NODE            DESIRED STATE  CURRENT STATE               ERROR  PORTS
t2709s75om3f  getstartedlab_web.1  dalpo/friendlyhello:v0.0.1  dalpo-thinkpad  Running        Running about a minute ago
3jovez76dtas  getstartedlab_web.2  dalpo/friendlyhello:v0.0.1  dalpo-thinkpad  Running        Running about a minute ago
tj315876iel0  getstartedlab_web.3  dalpo/friendlyhello:v0.0.1  dalpo-thinkpad  Running        Running about a minute ago
xinl1lmecg71  getstartedlab_web.4  dalpo/friendlyhello:v0.0.1  dalpo-thinkpad  Running        Running about a minute ago
57zppk2cn72s  getstartedlab_web.5  dalpo/friendlyhello:v0.0.1  dalpo-thinkpad  Running        Running about a minute ago
```

### Scale the app

You can scale the app by changing the replicas value in docker-compose.yml, saving the change, and re-running the `docker stack deploy` command:

```
± % docker stack deploy -c docker-compose.yml getstartedlab
Updating service getstartedlab_web (id: s8xxf37huej5cwp1jqw6bdywq)
```

### Take down the app

```
± % docker stack rm getstartedlab
Removing service getstartedlab_web
Removing network getstartedlab_webnet
```

## Swarm

Run `docker swarm init` to enable swarm mode and make your current machine a swarm manager.
Run `docker swarm join` on other machines to have them join the swarm as a worker.

Install `docker-machine`:

```
± % curl -L https://github.com/docker/machine/releases/download/v0.10.0/docker-machine-`uname -s`-`uname -m` > /tmp/docker-machine &&  chmod +x /tmp/docker-machine && sudo cp /tmp/docker-machine /usr/local/bin/docker-machine
```

Create a couple of VMs using docker-machine, using the VirtualBox driver:

```
$ # Create the manager
$ docker-machine create --driver virtualbox myvm1
$ # Create the worker
$ docker-machine create --driver virtualbox myvm2
```

Note: use `docker-machine ls` to find the VM IP adrress.

```
± % docker-machine ls
NAME    ACTIVE   DRIVER       STATE     URL                         SWARM   DOCKER        ERRORS
myvm1   -        virtualbox   Running   tcp://192.168.99.100:2376           v17.05.0-ce
myvm2   -        virtualbox   Stopped                                       Unknown
```

### Initialize a Swarm Manager into the myvm1:

```
± % docker-machine ssh myvm1 "docker swarm init --advertise-addr 192.168.99.100:2376"

Swarm initialized: current node (pg57yhis9fgcrehld4z8f10w5) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-1gtq47bhsrbtlx9c7tmfnswtuqo9s35dm30wrv811fbqjjkvfp-6a6zuicf9cq5kk8caaq4bpdij \
    192.168.99.100:2376

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

### Join the swarm into the myvm2:

Note: You have to use `PORT 2377`

```
docker-machine ssh myvm2 "docker swarm join \
  --token SWMTKN-1-1gtq47bhsrbtlx9c7tmfnswtuqo9s35dm30wrv811fbqjjkvfp-6a6zuicf9cq5kk8caaq4bpdij \
  192.168.99.100:2377"

This node joined a swarm as a worker.
```

##### Check the existing swarm nodes

Use ssh to connect to the (docker-machine ssh myvm1), and run `docker node ls` to view the nodes in this swarm:

```
% docker-machine ssh myvm1                                                                                                           !10167
                      ##         .
                ## ## ##        ==
             ## ## ## ## ##    ===
         /"""""""""""""""""\___/ ===
    ~~~ {~~ ~~~~ ~~~ ~~~~ ~~~ ~ /  ===- ~~~
         \______ o           __/
           \    \         __/
            \____\_______/
_                 _   ____     _            _
| |__   ___   ___ | |_|___ \ __| | ___   ___| | _____ _ __
| '_ \ / _ \ / _ \| __| __) / _` |/ _ \ / __| |/ / _ \ '__|
| |_) | (_) | (_) | |_ / __/ (_| | (_) | (__|   <  __/ |
|_.__/ \___/ \___/ \__|_____\__,_|\___/ \___|_|\_\___|_|
Boot2Docker version 17.05.0-ce, build HEAD : 5ed2840 - Fri May  5 21:04:09 UTC 2017
Docker version 17.05.0-ce, build 89658be
docker@myvm1:~$ docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS
893kulpfo94akitm4t9snzlgr *   myvm1               Ready               Active              Leader
ldr7xgp16gno1h2fvym7wb635     myvm2               Ready               Active
docker@myvm1:~$
```

Type `exit` to get back out of that machine.

Alternatively, you could simply run: `docker-machine ssh myvm1 "docker node ls"`

### Deploy your app on a cluster:

Copy the file docker-compose.yml you created in part 3 to the swarm manager myvm1’s home directory (alias: ~) by using the docker-machine scp command:

```
± % docker-machine scp docker-compose.yml myvm1:~
```

Now have `myvm1` use its powers as a swarm manager to deploy your app.

```
± % docker-machine ssh myvm1 "docker stack deploy -c docker-compose.yml getstartedlab"
Creating network getstartedlab_webnet
Creating service getstartedlab_web
```

Note: all the commands you used with `docker-machine ssh` have been distributed between both myvm1 and myvm2.


```
± % docker-machine ssh myvm1 "docker stack ps getstartedlab"
ID                  NAME                  IMAGE                        NODE                DESIRED STATE       CURRENT STATE                ERROR               PORTS
rdsopp6vhkai        getstartedlab_web.1   dalpo/friendlyhello:v0.0.1   myvm1               Running             Running about a minute ago
a56j5oghc66t        getstartedlab_web.2   dalpo/friendlyhello:v0.0.1   myvm2               Running             Running about a minute ago
yq3xsi65guzf        getstartedlab_web.3   dalpo/friendlyhello:v0.0.1   myvm2               Running             Running about a minute ago
```

### Add a new service

Update the `docker-compose.yml` file with:

```
version: "3"
services:
  web:
    image: dalpo/friendlyhello:v0.0.1
    deploy:
      replicas: 5
      restart_policy:
        condition: on-failure
      resources:
        limits:
          cpus: "0.1"
          memory: 50M
    ports:
      - "80:80"
    networks:
      - webnet

  visualizer:
    image: dockersamples/visualizer:stable
    ports:
      - "8080:8080"
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
    deploy:
      placement:
        constraints: [node.role == manager]
    networks:
      - webnet

networks:
  webnet:
```

Copy the `docker-compose.yml` file into the swarm manager:

```
docker-machine scp docker-compose.yml myvm1:~
```

Now just re-run the docker stack deploy command on the manager, and whatever services need updating will be updated:

```
$ docker-machine ssh myvm1 "docker stack deploy -c docker-compose.yml getstartedlab"
Updating service getstartedlab_web (id: r37meh4tc1v8lmcqm3qrr7z6f)
Updating service getstartedlab_visualizer (id: q6juheuv65v9vfqw0e8b6ow0u)
```

You saw in the Compose file that visualizer runs on port 8080. Get the IP address of the one of your nodes by running `docker-machine ls`. Go to either IP address @ port 8080 and you will see the visualizer running.

```
± % docker-machine ls
NAME    ACTIVE   DRIVER       STATE     URL                         SWARM   DOCKER        ERRORS
myvm1   -        virtualbox   Running   tcp://192.168.99.100:2376           v17.05.0-ce
myvm2   -        virtualbox   Running   tcp://192.168.99.101:2376           v17.05.0-ce
```

> Open the browser at https://192.168.99.100:8080

The single copy of visualizer is running on the manager as you expect, and the five instances of web are spread out across the swarm.

```
docker-machine ssh myvm1 "docker stack ps getstartedlab"
```

## Persisting data

Add a redis service with a persisted volume.

```
version: "3"
services:
  web:
    image: dalpo/friendlyhello:v0.0.1
    deploy:
      replicas: 5
      restart_policy:
        condition: on-failure
      resources:
        limits:
          cpus: "0.1"
          memory: 50M
    ports:
      - "80:80"
    networks:
      - webnet
  visualizer:
    image: dockersamples/visualizer:stable
    ports:
      - "8080:8080"
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
    deploy:
      placement:
        constraints: [node.role == manager]
    networks:
      - webnet
  redis:
    image: redis
    ports:
      - "6379:6379"
    volumes:
      - ./data:/data
    deploy:
      placement:
        constraints: [node.role == manager]
    networks:
      - webnet
networks:
  webnet:

```

Redis would store its data in `/data` inside the container’s filesystem, which would get wiped out if that container were ever redeployed.

```
$ docker-machine ssh myvm1 "mkdir ./data"
$ docker-machine scp docker-compose.yml myvm1:~
$ docker-machine ssh myvm1 "docker stack deploy -c docker-compose.yml getstartedlab"
```

# TROUBLESHOTTING:

- https://github.com/moby/moby/issues/23828
