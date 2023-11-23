<style>
.reveal h1, .reveal h2, .reveal h3, .reveal h4, .reveal h5 { text-transform: none; }
</style>

# Run any software<br>on NS8

---

<object data="NS8_Glossary_0.svg" type="image/svg+xml"></object>

---

<object data="NS8_Glossary_1.svg" type="image/svg+xml"></object>

---

<object data="NS8_Glossary_2.svg" type="image/svg+xml"></object>

---

<object data="NS8_Glossary_3.svg" type="image/svg+xml"></object>

---

<object data="NS8_Glossary_4.svg" type="image/svg+xml"></object>

---

<object data="NS8_Glossary_5.svg" type="image/svg+xml"></object>

---

<object data="NS8_Glossary_6.svg" type="image/svg+xml"></object>

---

<object data="NS8_Glossary_7.svg" type="image/svg+xml"></object>

---

<object data="NS8_Glossary_8.svg" type="image/svg+xml"></object>

---

<object data="NS8_Glossary_9.svg" type="image/svg+xml"></object>

---

What is running on the node?

```text [1,3]
[root@rl1 ~]# podman ps
CONTAINER ID  IMAGE                             COMMAND               CREATED            STATUS            PORTS       NAMES
99b96193930f  ghcr.io/nethserver/redis:2.2.0    redis-server /dat...  About an hour ago  Up About an hour              redis
449ff0f85f7f  ghcr.io/nethserver/restic:2.2.0   rclone serve webd...  About an hour ago  Up About an hour              rclone-webdav
f5b373c96005  docker.io/grafana/promtail:2.9.2  -config.file=/etc...  About an hour ago  Up About an hour              promtail
```

some **rootfull** containers<br/>
keep in mind Redis, on line 3 &#128070;

---

Again, what is running on the node?

```text [1,4]
[root@rl1 ~]# loginctl list-users
 UID USER        LINGER STATE
   0 root        no     active
1001 traefik1    yes    lingering
1002 scratchpad1 yes    lingering
1003 loki1       yes    lingering
1004 ldapproxy1  yes    lingering

5 users listed.
```

Unix user sessions<br/>
they are module instances (AKA _applications_)


---

`lingering`: started automatically at boot

---

what is running on the node?

# agents

- each module instance has one running agent
- started at boot, when the user session starts
- the agent runs actions and handles events

---


```text [1,3,4]
[root@rl1 ~]# ps -o user,cmd $(pgrep -x agent)
USER     CMD
root     /usr/local/bin/agent --agentid=cluster --actionsdir= --actionsdir=/var/lib/nethserver/cluster/actions --eventsdir=/var/lib/nethserver/cluster/events
root     /usr/local/bin/agent --agentid=node/1 --actionsdir= --actionsdir=/var/lib/nethserver/node/actions --eventsdir=/var/lib/nethserver/node/events
traefik1 /usr/local/bin/agent --agentid=module/traefik1 --actionsdir=/usr/local/agent/actions --actionsdir=/home/traefik1/.config/actions --eventsdir=/home/tra
scratch+ /usr/local/bin/agent --agentid=module/scratchpad1 --actionsdir=/usr/local/agent/actions --actionsdir=/home/scratchpad1/.config/actions --eventsdir=/ho
loki1    /usr/local/bin/agent --agentid=module/loki1 --actionsdir=/usr/local/agent/actions --actionsdir=/home/loki1/.config/actions --eventsdir=/home/loki1/.co
ldappro+ /usr/local/bin/agent --agentid=module/ldapproxy1 --actionsdir=/usr/local/agent/actions --actionsdir=/home/ldapproxy1/.config/actions --eventsdir=/home
```

A list of `agent` processes in the local node<br>
and the respective Unix user

---

<!-- .slide: style="text-align: left;"> -->  

&#11088; Special module-like agent contexts

- cluster
- node
- **rootfull modules** (optional)

...running as root

---

# agent environment

`runagent` runs a command as a module agent does

```text [1]
[root@rl1 ~]# runagent -m traefik1 whoami
traefik1
```

Does it recall `su` and `runuser`?

---

`runagent` in short
- runs command as a different user
- sets some environment variables


---

sets some environment variables,<br/>
like Redis credentials...

```text
[root@rl1 ~]# runagent -m traefik1 env | grep REDIS
REDIS_ADDRESS=cluster-leader:6379
REDIS_REPLICA_ADDRESS=127.0.0.1:6379
REDIS_USER=module/traefik1
REDIS_PASSWORD=4d3a5c36-f9c3-4c4c-b066-c794931d7c86
```

...and much more!

---

# where are<br>the containers?

---

Let's look at the Traefik instance

```text
[root@rl1 ~]# runagent -m traefik0 podman ps
[FATAL] Cannot find module traefik0 in the local node
```

OUCH! Bad module ID. List of valid module names?

```text [1,4]
[root@rl1 ~]# loginctl list-users
 UID USER        LINGER STATE
   0 root        no     active
1001 traefik1    yes    lingering
...
```

---

Let's look at the Traefik instance (fixed)

```text [1]
[root@rl1 ~]# runagent -m traefik1 podman ps
CONTAINER ID  IMAGE                           COMMAND     CREATED      STATUS      PORTS       NAMES
f08538d809a9  docker.io/library/traefik:v2.9  traefik     9 hours ago  Up 9 hours              traefik
```

---

Get a module shell, to run many commands

```text [1,2]
[root@rl1 ~]# runagent -m traefik1 bash -l
[traefik1@rl1 state]$ id
uid=1001(traefik1) gid=1001(traefik1) groups=1001(traefik1)
```

----

List available Systemd service units

```text [1,2,5]
[traefik1@rl1 state]$ systemctl --user --type=service -q
  agent.service                  loaded active running Rootless module/traefik1 agent
  dbus-broker.service            loaded active running D-Bus User Message Bus
  systemd-tmpfiles-setup.service loaded active exited  Create User's Volatile Files and Directories
  traefik.service                loaded active running Traefik edge proxy
```

----

Check `traefik.service` status

```text [1,3]
[traefik1@rl1 state]$ systemctl --user status traefik.service
● traefik.service - Traefik edge proxy
     Loaded: loaded (/home/traefik1/.config/systemd/user/traefik.service; enabled; preset: disabled)
     Active: active (running) since Thu 2023-11-23 08:43:29 UTC; 9h ago
    Process: 29941 ExecStartPre=/bin/rm -f /run/user/1001/traefik.pid /run/user/1001/traefik.ctr-id (code=exited, status=0/SUCCESS)
    Process: 29942 ExecStart=/usr/bin/podman run --detach --conmon-pidfile=/run/user/1001/traefik.pid --cidfile=/run/user/1001/traefik.ctr-id --cgroups=no-conmon --network=host --replace --name=traefik --volume=traefik-acme:/etc/traefik/acme --volume=./traefik.yaml:/etc/traefik/traefik.yaml:Z --volume=./selfsigned.crt:/etc/traefik/selfsigned.crt:Z --volume=./selfsigned.key:/etc/traefik/selfsigned.key:Z --volume=./configs:/etc/traefik/configs:Z --volume=./custom_certificates:/etc/traefik/custom_certificates:Z ${TRAEFIK_IMAGE} (code=exited, status=0/SUCCESS)
   Main PID: 29951 (conmon)
      Tasks: 1 (limit: 10864)
     Memory: 788.0K
        CPU: 212ms
     CGroup: /user.slice/user-1001.slice/user@1001.service/app.slice/traefik.service
             └─29951 /usr/bin/conmon --api-version 1 -c f08538d809a9eaf8ba080f19427494d3f0f1e9713847522029aa8385b42eac67 -u f08538d809a9eaf8ba080f19427494d3f0f1e9713847522029aa8385b42eac67 -r /usr/bin/crun -b /home/traefik1/.local/share/containers/storage/overlay-containers/f08538d809a9eaf8ba080f19427494d3f0f1e9713847522029aa8385b42eac67/userdata -p /run/user/1001/containers/overlay-containers/f08538d809a9eaf8ba080f19427494d3f0f1e9713847522029aa8385b42eac67/userdata/pidfile -n traefik --exit-dir /run/user/1001/libpod/tmp/exits --full-attach -s -l journald --log-level warning --syslog --runtime-arg --log-format=json --runtime-arg --log --runtime-arg=/run/user/1001/containers/overlay-containers/f08538d809a9eaf8ba080f19427494d3f0f1e9713847522029aa8385b42eac67/userdata/oci-log --conmon-pidfile /run/user/1001/traefik.pid --exit-command /usr/bin/podman --exit-command-arg --root --exit-command-arg /home/traefik1/.local/share/containers/storage --exit-command-arg --runroot --exit-command-arg /run/user/1001/containers --exit-command-arg --log-level --exit-command-arg warning --exit-command-arg --cgroup-manager --exit-command-arg systemd --exit-command-arg --tmpdir --exit-command-arg /run/user/1001/libpod/tmp --exit-command-arg --network-config-dir --exit-command-arg "" --exit-command-arg --network-backend --exit-command-arg netavark --exit-command-arg --volumepath --exit-command-arg /home/traefik1/.local/share/containers/storage/volumes --exit-command-arg --db-backend --exit-command-arg boltdb --exit-command-arg --transient-store=false --exit-command-arg --runtime --exit-command-arg crun --exit-command-arg --storage-driver --exit-command-arg overlay --exit-command-arg --events-backend --exit-command-arg file --exit-command-arg container --exit-command-arg cleanup --exit-command-arg f08538d809a9eaf8ba080f19427494d3f0f1e9713847522029aa8385b42eac67
```

---

Add a scratchpad instance

```text [1,4]
[root@rl1 ~]# add-module ghcr.io/davideprincipi/scratchpad:latest
<7>podman-pull-missing ghcr.io/davideprincipi/
...
{'module_id': 'scratchpad1', 'image_name': 'scratchpad', 'image_url': 'ghcr.io/davideprincipi/scratchpad:latest'} 
```
