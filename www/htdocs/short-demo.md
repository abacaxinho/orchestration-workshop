---

# VM environment

We have 5 Docker VMs:

- node (10.10.10.10)
- node (10.10.10.20)
- node (10.10.10.30)
- node (10.10.10.40)
- node (10.10.10.50)

---

## Brand new versions!

- Engine 1.12
- Compose 1.7
- Swarm 1.2
- Machine 0.6

Check all installed versions:
  ```bash
  docker version
  docker-compose -v
  ```

---

# Our sample application

- Visit the GitHub repository with all the materials of this 
  [workshop](https://github.com/abacaxinho/orchestration-workshop):

- The application is in the [dockercoins](
  https://github.com/abacaxinho/orchestration-workshop/tree/master/dockercoins)
  sub-directory

  ... and 4 other services, each in its own directory:

  - `rng` = web service generating random bytes
  - `hasher` = web service computing hash of POSTed data
  - `worker` = background process using `rng` and `hasher`
  - `webui` = web interface to watch progress

---

## Dockercoins structure
```
.
├── docker-compose.yml
├── hasher
│   ├── Dockerfile
│   └── hasher.rb
├── rng
│   ├── Dockerfile
│   └── rng.py
├── webui
│   ├── Dockerfile
│   ├── files
│   └── webui.js
└── worker
    ├── Dockerfile
    └── worker.py
```
---

## What's this application?

- It is a DockerCoin miner!

- No, you can't buy coffee with DockerCoins

- How DockerCoins works:

  - `worker` asks to `rng` to give it random bytes
  - `worker` feeds those random bytes into `hasher`
  - each hash starting with `0` is a DockerCoin
  - DockerCoins are stored in`redis`
  - `redis` is also updated every second to track speed
  - you can see the progress with the `webui`

---

## Running the application

Without further ado, let's start our application.

- Go to the `dockercoins` directory, in the cloned repo:
  ```bash
  cd ~/orchestration-workshop/dockercoins
  ```

- Use Compose to build and run all containers:
  ```bash
  docker-compose up
  ```

Compose tells Docker to build all container images (pulling
the corresponding base images), then starts all containers,
and displays aggregated logs.

---

## Lots of logs

- The application continuously generates logs
- We can see the `worker` service making requests to `rng` and `hasher`
- Let's put that in the background

- Stop the application by hitting `^C`

- `^C` stops all containers by sending them the `TERM` signal
- Some containers exit immediately, others take longer
  (because they don't handle `SIGTERM` and end up being killed after a 10s timeout)

---

## Restarting in the background

- Many flags and commands of Compose are modeled after those of `docker`
- Start the app in the background with the `-d` option:

  ```bash
  docker-compose up -d
  ```

- Check that our app is running with the `ps` command:

  ```bash
  docker-compose ps
  ```

`docker-compose ps` also shows the ports exposed by the application.

---

## Connecting to the web UI

- The `webui` container exposes a web dashboard; let's view it
  Open http://10.10.10.10:8000/ (from a browser)


- The app actually has a constant, steady speed (3.33 coins/second)
- The speed seems not-so-steady because:
- the worker doesn't update the counter after every loop, but up to once per second
- the speed is computed by the browser, checking the counter about once per second
- between two consecutive updates, the counter will increase either by 4, or by 0

---

## Viewing logs

- The `docker-compose logs` command works like `docker logs`
- View all logs since container creation and exit when done:
  ```bash
  docker-compose logs
  ```

- Stream container logs, starting at the last 10 lines for each container:
  ```bash
  docker-compose logs --tail 10 --follow
  ```
  Tip: use `^S` and `^Q` to pause/resume log output.

---

## Testing services in isolation

- We will stop the `worker` service, and test `rng` and `hasher` alone

- Stop the `worker` service:
  ```bash
  docker-compose stop worker
  ```

- Look at the logs of `rng` and `hasher`:
  ```bash
  docker-compose logs --tail 10 --follow
  ```

---

## Compose file location

- What happens if you try to run Compose in `$HOME`?

- Compose complains that it cannot find `docker-compose.yml`
- You need to go to the directory containing `docker-compose.yml`
  ...or use the `-f` option to give the path to the Compose file
  ...or set the `COMPOSE_FILE` environment variable

- Go to the application directory:
  ```bash
  cd ~/orchestration-workshop/dockercoins
  ```

---

## Locating port numbers for `rng` and `hasher`

- We can see the port mapping for our services with:
  ```bash
  docker-compose ps
  ```

- Both services run on port 80 *in their respective container*
- On the Docker host, they are mapped to ports 8001 and 8002
- They are not exposed on port 80, because the Docker host has only one port 80
- The mapping is done in `docker-compose.yml`

---

# Container port mapping

- `node1`, the Docker host, has only one port 80

- If we give the one and only port 80 to the first
  container who asks for it, we are in trouble when
  another container needs it

- Default behavior: containers are not "exposed"
  (only reachable by the Docker host and other containers,
  through their private address)

- Container network services can be exposed:
  - statically (you decide which host port to use)
  - dynamically (Docker allocates a host port)

---

## Declaring port mapping

- Directly with the Docker Engine:
  ```bash
  docker run -d -p 8000:80 nginx
  docker run -d -p 80 nginx
  docker run -d -P nginx
  ```

- With Docker Compose, in the `docker-compose.yml` file:
  ```yaml
    rng:
      …
      ports:
        - "8001:80"
  ```

- port 8001 *on the host* maps to port 80 *in the container*

---

## Testing the `rng` service

Let's get random bytes of data!

- Check that the service is alive:
  `curl localhost:8001`

- Get 10 bytes of random data:
  `curl localhost:8001/10`

If the binary data output messed up your terminal, fix it with `reset`.

---

## Testing the `hasher` service

- Check that the `hasher` service is alive:
  ```bash
  curl localhost:8002
  ```

- Posting binary data requires some extra flags:
  ```bash
    curl -H "Content-type: application/octet-stream" \
      --data-binary hello localhost:8002
  ```

---

## Verifying the computed hash

- This web service is essentially implementing `sha256sum` over HTTP

- Check that `sha256sum` yields the same result:
  ```bash
  echo -n hello | sha256sum
  ```

- We specify `-n` to prevent `echo` from adding a trailing `\n`
  (which would change the SHA digest)

---

## Checking logs

- The tests that we made should show up in the first window
  (where `docker-compose logs` is still running)

- We can now restart the `worker` service

- Start the `worker` container:
  ```bash
  docker-compose start worker
  ```

In the web UI, the graph should go up again.
How can we get that graph to go further up?

---

## Looking at resource usage

- Let's look at CPU, memory, and I/O usage
- run `top` to see CPU and memory usage (you should see idle cycles)
- run `vmstat 3` to see I/O usage (si/so/bi/bo)
  (the 4 numbers should be almost zero, except `bo` for logging)

We have available resources.

- Why?
- How can we use them?

---

## Scaling workers on a single node

- Docker Compose supports scaling
- Let's scale `worker` and see what happens!

- Start 9 more `worker` containers:
  ```bash
  docker-compose scale worker=10
  ```

- Check the aggregated logs of those containers
- See the impact on CPU load (with top/htop), and on compute speed (with web UI)

```bash
sleep 5
killall docker-compose
```

---
NOTE: cia praskipinam visa jerunda apie sleep()
---

## The plan

- Stop the `rng` service first
- Create multiple identical `rng` containers
- Put a load balancer in front of them
- Point other services to the load balancer

---

## Stopping `rng`

- That's the easy part!

- Use `docker-compose` to stop `rng`:
  ```bash
  docker-compose stop rng
  ```

Note: we do this first because we are about to remove
`rng` from the Docker Compose file.

If we don't stop
`rng` now, it will remain up and running, with Compose
being unaware of its existence!

---

## Scaling `rng`

- Replace the `rng` service with multiple copies of it:
  ```yaml
    rng1:
      build: rng

    rng2:
      build: rng

    rng3:
      build: rng
  ```

That's all!

Shortcut: `docker-compose.yml-scaled-rng`

---

## Introduction to `jpetazzo/hamba`

- General purpose load balancer and traffic director
- [Source code is available on GitHub](
  https://github.com/jpetazzo/hamba)
- [Public image is available on the Docker Hub](
  https://hub.docker.com/r/jpetazzo/hamba/)
- Generates a configuration file for HAProxy, then starts HAProxy
- Parameters are provided on the command line; for instance:
  ```bash
  docker run -d -p 80 jpetazzo/hamba 80 www1:1234 www2:2345
  docker run -d -p 80 jpetazzo/hamba 80 www1 1234 www2 2345
  ```

  Those two commands do the same thing: they start a load balancer
  listening on port 80, and balancing traffic across www1:1234 and www2:2345

---

# Put a load balancer on it

Let's add our load balancer to the Compose file.

- Add the following section to the Compose file:
  ```yaml
  rng:
      image: jpetazzo/hamba
      links:
        - rng1
        - rng2
        - rng3
      command: 80 rng1 80 rng2 80 rng3 80
      ports:
        - "8001:80"
  ```

Shortcut: `docker-compose.yml-scaled-rng`

---

## Start the whole stack

- Start the new services:
  `docker-compose up -d`

- Check worker logs:
  `docker-compose logs worker`

- Check load balancer logs:
  `docker-compose logs rng`

<!--
If you get errors about port 8001, make sure that
`rng` was stopped correctly and try again.
-->

---

## Results

- Check the latency of `rng`
  (it should have improved significantly!)

- Check the application performance in the web UI
  (it should improve if you have enough workers)

*Note: if `worker` was scaled when you did `docker-compose up`,
it probably took a while, because `worker` doesn't handle
signals properly and Docker patiently waits 10 seconds for
each `worker` instance to terminate. This would be much
faster for a well-behaved application.*

---

## The good, the bad, the ugly

- The good
  We scaled a service, added a load balancer - without changing a single line of code.

- The bad
  We manually copy-pasted sections in `docker-compose.yml`.
  Improvement: write scripts to transform the YAML file.

- The ugly
  If we scale up/down, we have to restart everything.
  Improvement: reconfigure the load balancer dynamically.

---

## Cleaning up

- Before moving forward, let's clean up all those containers

- Use the `down` command to stop and remove containers:
  ```bash
  docker-compose down
  ```

---

# Connecting to containers on other hosts

- So far, our whole stack is on a single machine
- We want to scale out (across multiple nodes)
- We will deploy the same stack multiple times
- But we want every stack to use the same Redis
  (in other words: Redis is our only *stateful* service here)

- We're not allowed to change the code!
  - the code connects to host `redis`
  - `redis` must resolve to the address of our Redis service
  - the Redis service must listen on the default port (6379)

---

## Using host name injection to abstract service dependencies

- It is possible to add host entries to a container
- With the CLI:
  ```bash
  docker run --add-host redis:192.168.1.2 myservice...
  ```

- In a Compose file:
  ```yaml
  myservice:
    image: myservice
    extra_host:
      redis: 192.168.1.2
  ```

- Docker exposes a DNS server to the container,
  with a private view where `redis` resolves to `192.168.1.2`
  (Before Engine 1.10, it created entries in `/etc/hosts`)

---

## The plan

- Deploy our Redis service separately
  - use the same `redis` image
  - make sure that Redis server port (6379) is publicly accessible,
    using port 6379 on the Docker host
- Update our Docker Compose YAML file
  - remove the `redis` section
  - in the `links` section, remove `redis`
  - instead, put a `redis` entry in `extra_hosts`

Note: the code stays on the first node!
(We do not need to copy the code to the other nodes.)

---

## Making Redis available on its default port

There are two strategies.

- `docker run -p 6379:6379 redis`
  - the container has its own, isolated network stack
  - Docker creates a port mapping rule through iptables
  - slight performance overhead
  - port number is explicit (visible through Docker API)
- `docker run --net host redis`
  - the container uses the network stack of the host
  - when it binds to 6379/tcp, that's 6379/tcp on the host
  - allows raw speed (no overhead due to iptables/bridge)
  - port number is not visible through Docker API

Choose wisely!

---

## Deploy Redis

- Start a new redis container, mapping port 6379 to 6379:
  ```bash
  docker run -d -p 6379:6379 redis
  ```

- Check that it's running with `docker ps`
- Note the IP address of this Docker host
- Try to connect to it (from anywhere):
  ```bash
  curl node1:6379
  ```
The `ERR` messages are normal: Redis speaks Redis, not HTTP.

---

## Update `docker-compose.yml` (1/3)

- Comment out `redis`:
  ```yaml
  #redis:
  #    image: redis
  ```
---

## Update `docker-compose.yml` (2/3)

- Update `worker`:
  ```yaml
  worker:
    build: worker
    extra_hosts:
      redis: AA.BB.CC.DD
    links:
      - rng
      - hasher
  ```

Replace `AA.BB.CC.DD` with the IP address noted earlier.

Shortcut: `docker-compose.yml-extra-hosts`
(But you still have to replace `AA.BB.CC.DD`!)

---

## Update `docker-compose.yml` (3/3)

- Update `webui`:
  ```yaml
  webui:
    build: webui
    extra_hosts:
      redis: AA.BB.CC.DD
    ports:
      - "8000:80"
    volumes:
      - "./webui/files/:/files/"
  ```

(Replace `AA.BB.CC.DD` with the IP address noted earlier)

---

## Start the stack on the first machine

- Nothing special to do here
- Just bring up the application like we did before
- `docker-compose up -d`

---

## Start the stack on another machine

- We will set the `DOCKER_HOST` variable
- `docker-compose` will detect and use it
- Our Docker hosts are listening on port 55555

- Set the environment variable:
  `export DOCKER_HOST=tcp://node2:55555`

- Start the stack:
  `docker-compose up -d`

- Check that it's running:
  `docker-compose ps`


---

## Scale!

- Keep an eye on the web UI
- Create 20 workers on both nodes:
  ```bash
  for NODE in node1 node2; do
    export DOCKER_HOST=tcp://$NODE:55555
    docker-compose scale worker=20
  done
  ```

Note: of course, if we wanted, we could run on all five nodes.

---

## Cleanup

- Let's remove what we did
- You can use the following scriptlet:
  ```bash
  for N in $(seq 1 5); do
    export DOCKER_HOST=tcp://node$N:55555
    docker ps -qa | xargs docker rm -f
  done
  unset DOCKER_HOST
  ```

---

# Using custom DNS mapping

- We could setup a Redis server on its default port
- And add a DNS entry mapping `redis` to this server
- See what happens if we run:
  ```bash
  docker run --add-host redis:1.2.3.4 alpine ping redis
  ```

There is a Compose file option for that: `extra_hosts`.

---

# Abstracting remote services with ambassadors

- What if we can't/won't run Redis on its default port?
- What if we want to be able to move it easily?

- We will use an ambassador
- Redis will be started independently of our stack
- It will run at an arbitrary location (host+port)
- In our stack, we replace `redis` with an ambassador
- The ambassador will connect to Redis
- The ambassador will "act as" Redis in the stack

---

class: pic

![Ambassador principle](ambassadors/principle-2.png)

---

## Start redis

- Start a standalone Redis container
- Let Docker expose it on a random port

- Run redis with a random public port:
  `docker run -d -P --name myredis redis`

- Check which port was allocated:
  `docker port myredis 6379`


- Note the IP address of the machine, and this port

---

## Update `docker-compose.yml`

- Replace `redis` with an ambassador using `jpetazzo/hamba`:
  ```yaml
    redis:
      image: jpetazzo/hamba
      command: 6379 `AA.BB.CC.DD EEEEE`
  ```

Shortcut: `docker-compose.yml-ambassador`
(But you still have to update `AA.BB.CC.DD EEEEE`!)

---

## Start the stack on the first machine

- Compose will detect the change in the `redis` service
- It will replace `redis` with a `jpetazzo/hamba` instance

- Just tell Compose to do its thing:
  `docker-compose up -d`

- Check that the stack is up and running:
  `docker-compose ps`

- Look at the web UI to make sure that it works fine

---

## Controlling other Docker Engines

- Many tools in the ecosystem will honor the `DOCKER_HOST` environment variable
- Those tools include (obviously!) the Docker CLI and Docker Compose
- Our training VMs have been setup to accept API requests on port 55555
  (without authentication - this is very insecure, by the way!)

- Set the `DOCKER_HOST` variable to `node2`, and execute a `docker` command:
  ```bash
  export DOCKER_HOST=tcp://node2:55555
  docker ps
  ```

You shouldn't see any container running at this point.

---

## Start the stack on another machine

- We will tell Compose to bring up our stack on the other node
- It will use the local code (we don't need to checkout the code on `node2`)

- Start the stack:
  ```bash
  docker-compose up -d
  ```

- Check that it's running:
  ```bash
  docker-compose ps
  ```

---

## Run the application on every node

- We will repeat the previous step with a little shell loop
  ... but introduce parallelism to save some time


- Deploy one instance of the stack on each node:
  ```bash
    for N in 3 4 5; do
      DOCKER_HOST=tcp://node$N:55555 docker-compose up -d &
    done
    wait
  ```

Note: building the stack everywhere is not optimal. We will see later
how to build once, and deploy the same build everywhere.

---

## Scale!

- The app is built (and running!) everywhere
- Scaling can be done very quickly

- Add a bunch of workers all over the place:
  ```bash
    for N in 1 2 3 4 5; do
      DOCKER_HOST=tcp://node$N:55555 docker-compose scale worker=10
    done
  ```

- Admire the result in the web UI!

