# Tutorial: A Mad Scientist's Guide to Automating CNI Configurations using Generative AI

A kubecon tutorial about automating CNI configuration using an LLM

From [this kubecon tutorial session](https://kccncna2024.sched.com/event/1i7kI?iframe=no):

> Ready to make Kubernetes networking a little easier and a lot more fun? Join Doug for an experiment in configuring CNI (Container Networking Interface) using generative AI. Despite being advised by data scientists to avoid automating machine configurations with generative AI, Doug went into the mad scientist's lab (err, basement) and tested how often a workflow could generate CNI configurations that would establish network connectivity between pods – and the success rate might surprise you. In this session, you'll automate CNI configurations using a large language model (LLM) and gain experience with a nifty tech stack: Ollama for running a containerized LLM, Kubernetes, CNI, and some script wizardry to create your own auto-configurator. Best yet? No prior CNI or AI/ML knowledge needed, and you'll learn along the way! Just in case, have contingency plans ready should any Skynet or Space Odyssey 2001 scenarios arise during the tutorial.

The recording is available [here](https://youtu.be/tEVoXKtymIA?si=rHv_5ujUM2cbej70).

## Goal

Spin up a [KIND (kubernetes in Docker)](https://kind.sigs.k8s.io/) cluster that uses [CNI (container network interface)](https://github.com/containernetworking) configurations generated from an LLM, and we'll test to see if there's connectivity over that network.

## Demo recording on asciinema

[![asciicast](https://asciinema.org/a/687857.svg)](https://asciinema.org/a/687857)

## Requirements!

I'll be using a Fedora 40 system, but, you can use anything that's capable of these requirements:

* A linux (or linux-like system) that's capable of installing KIND
* Docker
* Git (potentially optional)

(Sorry I didn't have a mac to test this on, it might be a bit of a toss up)

## Bonus requirements

* A machine with a GPU!

## Step 1: Install robocniconfig

Let's install: https://github.com/dougbtv/robocniconfig

We can install it via binaries, if you're using linux amd 64 type architecure....

```
curl -L -o robocni https://github.com/dougbtv/robocniconfig/releases/download/v0.0.2/robocni
curl -L -o looprobocni https://github.com/dougbtv/robocniconfig/releases/download/v0.0.2/looprobocni
chmod +x robocni
chmod +x looprobocni
sudo mv looprobocni /usr/local/bin/
sudo mv robocni /usr/local/bin/
robocni -help
```

### OPTIONAL: Build `robocniconfig` with golang

Get a golang environment going so we can build it...

The lastest install instructions are here: https://go.dev/doc/install

```
curl -L -o go1.23.2.linux-amd64.tar.gz https://go.dev/dl/go1.23.2.linux-amd64.tar.gz
rm -rf /usr/local/go && tar -C /usr/local -xzf go1.23.2.linux-amd64.tar.gz
export PATH=$PATH:/usr/local/go/bin
go version
```

Now that we have that, we can clone the robocni

```
git clone https://github.com/dougbtv/robocniconfig.git 
cd robocniconfig/
```

And you can run the buildscript:

```
./hack/build-go.sh
```

And add the path of those binaries to our path...

```
export PATH=$PATH:$(pwd)/bin
```

And make sure you can run it:

```
robocni --help
```

## Step 2a: If need be, install docker

Installation steps for Fedora shown

```
sudo dnf -y install dnf-plugins-core
sudo dnf-3 config-manager --add-repo https://download.docker.com/linux/fedora/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl start docker
sudo systemctl enable docker
sudo groupadd docker
sudo usermod -aG docker $USER
newgrp docker
docker ps
```

## Step 2b: Install KIND

So, KIND is Kubernetes-in-Docker! It's a simple way to run a cluster without needing large infra, great for test environments.

Install steps @ https://kind.sigs.k8s.io/docs/user/quick-start/#installing-from-release-binaries

Fedora install steps shown, download it:

```
[ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.24.0/kind-linux-amd64
```

Then, install it:

```
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
kind version
```

And install kubectl! If you don't have it.

```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
kubectl version
```

## Step 3: Install required tools

We're going to use koko: https://github.com/redhat-nfvpe/koko

"The container connector" -- since our KIND nodes are emulated, they're just docker containers, we need a way to virtually connect them, which we will do with this tool to create virtual interfaces in the host containers for KIND.

And we'll install it with:

```
sudo curl -Lo /bin/koko https://github.com/redhat-nfvpe/koko/releases/download/v0.83/koko_0.83_linux_amd64
sudo chmod +x /bin/koko
koko version
```

(`koko version` will just exit 0, no output)

## Step 4: Configure KIND and spin up a cluster

Spin up a kind cluster using this yaml...

```
cat <<EOF | kind create cluster --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
EOF
```

Now we wait for the image to download, and the nodes to start...

And we can do:

```
kubectl get nodes
```

## Step 4a: *(FULLY OPTIONAL)* CNI CHALLENGE MODE.

*Doug will skip this during the tutorial.*

If you're brave, you could instead create a cluster and spin up flannel, an alternative CNI.

It will probably decrease your success rates.

```
cat <<EOF | kind create cluster --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
networking:
  disableDefaultCNI: true
  podSubnet: 10.244.0.0/16
EOF
```

First, we disabled the default CNI, so we'll need to install our own.

You can see that the nodes aren't ready yet, this is a CNI thing.

```
kubectl get nodes
```

We're going to install [Flannel](https://github.com/flannel-io/flannel)

It requires that we have the `br_netfiler` kernel module loaded, which we'll have to do, you can do the nodes one at a time like this:

```
docker exec -it kind-worker2 modprobe br_netfilter
```

Or, all of them at once with:

```
kubectl get nodes | grep -v "NAME" | awk '{print $1}' | xargs -I {} docker exec -i {} modprobe br_netfilter
```

So let's apply that and wait for it.

```
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/refs/heads/master/Documentation/kube-flannel.yml
kubectl wait --for=jsonpath='{.status.numberReady}'=$(kubectl get daemonset kube-flannel-ds -n kube-flannel -o jsonpath='{.status.desiredNumberScheduled}') daemonset/kube-flannel-ds -n kube-flannel --timeout=5m
```

And we can see the nodes are ready

```
kubectl get nodes
```


## Step 5: Base CNI configuration for your nodes

We are going to install Multus CNI: 

Multus enables us to attach multiple network interfaces to pods.

```
kubectl create -f https://raw.githubusercontent.com/k8snetworkplumbingwg/multus-cni/refs/heads/master/deployments/multus-daemonset-thick.yml
kubectl -n kube-system wait --for=condition=ready -l name=multus pod --timeout=300s
```

We need to install the reference CNI plugins, e.g. https://github.com/containernetworking/plugins

```
kubectl create -f https://raw.githubusercontent.com/k8snetworkplumbingwg/multus-cni/refs/heads/master/e2e/templates/cni-install.yml.j2
kubectl -n kube-system wait --for=condition=ready -l name=cni-plugins pod --timeout=300s
```

And we'll install whereabouts, an IPAM CNI plugin

```
kubectl create -f https://raw.githubusercontent.com/k8snetworkplumbingwg/whereabouts/refs/heads/master/doc/crds/daemonset-install.yaml -f https://raw.githubusercontent.com/k8snetworkplumbingwg/whereabouts/refs/heads/master/doc/crds/whereabouts.cni.cncf.io_overlappingrangeipreservations.yaml -f https://raw.githubusercontent.com/k8snetworkplumbingwg/whereabouts/refs/heads/master/doc/crds/whereabouts.cni.cncf.io_ippools.yaml
kubectl -n kube-system wait --for=condition=ready -l name=whereabouts pod --timeout=300s
```

Now we can see what these commands did, our CNI configuration is modified:

```
docker exec -it kind-worker cat /etc/cni/net.d/00-multus.conf && echo
```

And... we have a bunch of CNI plugins loaded...

```
docker exec -it kind-worker ls -l /opt/cni/bin
```

### Optional: Spin up a test pod

First we create a network attachment to define an additional network to attach to...

```
cat <<EOF | kubectl create -f -
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: macvlan-conf
spec:
  config: '{
      "cniVersion": "0.3.0",
      "type": "macvlan",
      "master": "eth0",
      "mode": "bridge",
      "ipam": {
        "type": "host-local",
        "subnet": "192.168.1.0/24",
        "rangeStart": "192.168.1.200",
        "rangeEnd": "192.168.1.216",
        "routes": [
          { "dst": "0.0.0.0/0" }
        ],
        "gateway": "192.168.1.1"
      }
    }'
EOF
```

And you can `kubectl get net-attach-def` to see it.

And then a sample pod...

```
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: samplepod
  annotations:
    k8s.v1.cni.cncf.io/networks: macvlan-conf
spec:
  containers:
  - name: samplepod
    command: ["/bin/ash", "-c", "trap : TERM INT; sleep infinity & wait"]
    image: alpine
EOF
```

And then we check if it has multiple interfaces:

```
kubectl exec -it samplepod -- ip a
```

## Step 6: Add an extra interface with koko! (possibly optional)

First inspect the interfaces in your host containers...

```
docker exec -it kind-worker ip a
```

Then, we'll tell koko to create a [veth](https://developers.redhat.com/blog/2018/10/22/introduction-to-linux-interfaces-for-virtual-networking#veth) between these two containers, in a fashion that looks like a linux interface...

```
worker1_pid=$(docker inspect --format "{{ .State.Pid }}" kind-worker)
worker2_pid=$(docker inspect --format "{{ .State.Pid }}" kind-worker2)
sudo koko -p "$worker1_pid,eth1" -p "$worker2_pid,eth1"
```

And we'll see we have an eth1, now.

```
docker exec -it kind-worker ip a
```

## Step 6: Choose your own adventure! Install an LLM (or use one I provide)

Choose:

* Spin up your own LLM with ollama
* Use one that Doug provides (a cloud instance)

### (Step 6) You chose: Install ollama yourself

You can install ollama yourself:

https://ollama.com/download

Linux shown, just with:

```
curl -fsSL https://ollama.com/install.sh | sh
```

I usually run ollama-serve in a screen...

```
screen -S ollama-serve
OLLAMA_HOST=0.0.0.0:8080 ollama serve
```

Then Hit `CTRL+a` (let go), then `d`

You can return to it with:

```
screen -r ollama-serve
```

Then a screen for run

```
screen -S ollama-run
OLLAMA_HOST=0.0.0.0:8080 ollama run codegemma:7b
```

Or run a different model, like so:

```
OLLAMA_HOST=0.0.0.0:8080 ollama run deepseek-coder-v2:16b
OLLAMA_HOST=0.0.0.0:8080 ollama run llama2:13b
```

### (Step 6) You chose: Use Doug's cloud provided Ollama instance

*Doug will give you an IP address and a port!*

IP ADDRESS: (stub)
PORT: (stub)

## Step 7: Run `robocniconfig`

Now, you can run robocniconfig itself!

First, let's export the values for our host and port

```
export OHOST=66.114.112.70
export OPORT=56673
export MODEL=llama3.1:70b
```

And now we'll query it

```
robocni -host $OHOST -model $MODEL -port $OPORT "give me a macvlan CNI configuration mastered to eth0 using host-local ipam ranged on 192.0.2.0/24" && echo
robocni -host $OHOST -model $MODEL -port $OPORT "give me a macvlan CNI config with ipam on 10.0.2.0/24 " && echo
```

Add the `-debug` flag if you're having problems. (It won't give you much, but, it might give you something)


```
robocni -host $OHOST -model $MODEL -port $OPORT "name a macvlan configuration after a historical event in science" && echo
```

Now let's create a `promptfile` where we'll put a series of prompts we want to test...

```
give me a macvlan CNI configuration mastered to eth0 using whereabouts ipam ranged on 192.0.2.0/24, give it a whimsical name for children
an ipvlan configuration on eth0 with whereabouts for 10.40.0.15/27 named after a street in brooklyn
type=macvlan master=eth0 whereabouts=10.30.0.0/24 name~=$(after a significant landmark)
ipvlan for eth0, ipam is whereabouts on 192.168.50.100/28 exclude 192.168.50.101/32
dude hook me up with a macvlan mastered to eth0 with whereabouts on a 10.10.0.0/16
macvlan eth0 whereabouts 10.40.0.0/24 name geographical
macvlan on whereabouts named after a US president
ipvlan on eth1 named after a random fruit
```

Put these contents of this in a file, I put mine in `/tmp/prompts.txt`


```
looprobocni -host $OHOST -model $MODEL -introspect -port $OPORT -promptfile /tmp/prompts.txt
```

Now we can run it for 5 runs...

```
looprobocni -host $OHOST -model $MODEL -introspect -port $OPORT -promptfile /tmp/prompts.txt --runs 5
```
