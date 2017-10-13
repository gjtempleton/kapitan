# Kapitan

[![Build Status](https://travis-ci.org/deepmind/kapitan.svg?branch=master)](https://travis-ci.org/deepmind/kapitan)

a tool to manage kubernetes configuration using jsonnet templates

Kapitan is a command line tool for declaring, instrumenting and documenting
infrastructure with the goal of writing reusable components in Kubernetes whilst avoiding
duplication and promoting conventions and patterns for extensibility.



# Table of Contents

* [Kapitan](#kapitan)
* [Table of Contents](#table-of-contents)
* [Main Features](#main-features)
* [Installation](#installation)
* [Example](#example)
* [Main concepts](#main-concepts)
* [Typical folder structure](#typical-folder-structure)
* [Usage](#usage)
* [Modes of operation](#modes-of-operation)
* [Credits](#credits)
* [Related projects](#related-projects)



# Main Features

* Create kubernetes manifests templates using [Jsonnet](https://github.com/google/jsonnet)
* Create templates for scripts and documentation using [Jinja2](http://jinja.pocoo.org/docs/2.9/)
* Designed to work with multiple targets / environments
* Define variables in a hierarchical tree using [reclass](https://github.com/madduck/reclass)



# Installation

Kapitan needs Python 2.7+ and can be installed with pip.

```
$ pip install git+https://github.com/deepmind/kapitan.git
```



# Example

The example below _compiles_ a target inside the `examples` folder called `minikube-es`.
This target generates the following resources:

* Kubernetes `StatefulSet` for ElasticSearch Master node
* Kubernetes `StatefulSet` for ElasticSearch Client node
* Kubernetes `StatefulSet` for ElasticSearch Data node
* Kubernetes `Service` to expose ElasticSearch discovery port
* Kubernetes `Service` to expose ElasticSearch service port
* Script `setup.sh` to configure kubectl context for this target
* Script `kubectl.sh` to control this target
* Documentation

![demo](https://raw.githubusercontent.com/deepmind/kapitan/master/docs/demo.gif)

```
$ cd examples/

$ kapitan compile -f targets/minikube-es/target.json
Wrote compiled/minikube-es/manifests/es-master.yml
Wrote compiled/minikube-es/manifests/es-elasticsearch-svc.yml
Wrote compiled/minikube-es/manifests/es-discovery-svc.yml
Wrote compiled/minikube-es/manifests/es-client.yml
Wrote compiled/minikube-es/manifests/es-data.yml
Wrote compiled/minikube-es/scripts/setup.sh with mode 0740
Wrote compiled/minikube-es/scripts/kubectl.sh with mode 0740
Wrote compiled/minikube-es/docs/README.md with mode 0640
```



# Main concepts

### Targets

A target usually represents a single namespace in a kubernetes cluster and defines all components, scripts and documentation that will be generated for that target.

Kapitan requires a target file to compile a target and its parameters. Example:

```
$ cat examples/targets/minikube-es/target.yml
---
version: 1
vars:
  target: minikube-es
  namespace: minikube-es
compile:
  - name: manifests
    type: jsonnet
    path: targets/minikube-es/main.jsonnet
    output: yaml
  - name: scripts
    type: jinja2
    path: scripts
  - name: docs
    type: jinja2
    path: targets/minikube-es/docs
```

### Components

A component is an aplication that will be deployed to a kubernetes cluster. This includes all necessary kubernetes objects (Statefull set, services, configmaps) defined in jsonnet.
It may also include scripts, config files and dinamically generated documentation defined using Jinja templates.


### Inventory

This is a hierarchical database of variables that are passed to the targets during compilation.

By default, Kapitan will look for an `inventory/` directory to render the inventory from.

There are 2 types of objects inside the inventory:


#### Inventory Classes

Classes define variables that are shared across many targets. You can have for example a `component.elasticsearch` class with all the default values for targets using elasticsearch. Or a `production` or `dev` class to enable / disable certain features based on the type of target.

You can always override values further up the tree (i.e. in the inventory target file or in a class that inherits another class)

Classifying almost anything will help you avoid repetition (DRY) and will force you to organise parameters hierarchically.

For example, the snipppet below, taken from the example elasticsearch class, declares
what parameters are needed for the elasticsearch component:

```
$ cat inventory/classes/component/elasticsearch.yml
parameters:
  elasticsearch:
    image: "quay.io/pires/docker-elasticsearch-kubernetes:5.5.0"
    java_opts: "-Xms512m -Xmx512m"
    replicas: 1
    roles:
      master:
        image: ${elasticsearch:image}
        java_opts: ${elasticsearch:java_opts}
        replicas: ${elasticsearch:replicas}
...
```

#### Inventory Targets

Inside the inventory target files you can include classes and define new values or override any values inherited from the inncluded classes. For example:

```
$ cat inventory/targets/minikube-es.yml
classes:
  - cluster.minikube
  - component.elasticsearch

parameters:
  namespace: minikube-es

  elasticsearch:
    replicas: 2
```


# Typical folder structure

```
.
├── components
│   ├── elasticsearch
│   │   ├── configmap.jsonnet
│   │   ├── deployment.jsonnet
│   │   ├── main.jsonnet
│   │   └── service.jsonnet
│   └── nginx
│       ├── configmap.jsonnet
│       ├── deployment.jsonnet
│       ├── main.jsonnet
│       ├── nginx.conf.j2
│       └── service.jsonnet
├── inventory
│   ├── classes
│   │   ├── cluster
│   │   │   ├── cluster1.yml
│   │   │   └── cluster2.yml
│   │   ├── component
│   │   │   ├── elasticsearch.yml
│   │   │   ├── nginx.yml
│   │   │   └── zookeeper.yml
│   │   └── environment
│   │       ├── dev.yml
│   │       └── prod.yml
│   └── targets
│       ├── dev-cluster1-elasticsearch.yml
│       ├── prod-cluster1-elasticsearch.yml
│       └── prod-cluster2-frontend.yml
├── lib
│   ├── kapitan.libjsonnet
│   └── kube.libjsonnet
└── targets
    ├── dev-cluster1-elasticsearch
    │   ├── main.jsonnet
    │   └── target.yaml
    ├── prod-cluster1-elasticsearch
    │   ├── main.jsonnet
    │   └── target.yaml
    └── prod-cluster2-frontend
        ├── main.jsonnet
        └── target.yaml
```

# Usage

```
$ kapitan -h
usage: kapitan [-h] [--version] {eval,compile,inventory,searchvar} ...

Kapitan is a tool to manage kubernetes configuration using jsonnet templates

positional arguments:
  {eval,compile,inventory,searchvar}
                        commands
    eval                evaluate jsonnet file
    compile             compile target files
    inventory           show inventory
    searchvar           show all inventory files where var is declared

optional arguments:
  -h, --help            show this help message and exit
  --version             show program's version number and exit
```

Additional parameters are available for each positional argument. For example:

```
$ kapitan compile -h
usage: kapitan compile [-h] [--target-file TARGET [TARGET ...]]
                       [--search-path JPATH] [--verbose] [--no-prune]
                       [--quiet] [--output-path PATH] [--parallelism INT]

optional arguments:
  -h, --help            show this help message and exit
  --target-file TARGET [TARGET ...], -f TARGET [TARGET ...]
                        target files
  --search-path JPATH, -J JPATH
                        set search path, default is "."
  --verbose, -v         set verbose mode
  --no-prune            do not prune jsonnet output
  --quiet               set quiet mode, only critical output
  --output-path PATH    set output path, default is "./compiled"
  --parallelism INT, -p INT
                        Number of concurrent compile processes, default is 4
```



# Modes of operation

### kapitan eval


### kapitan compile

#### Using the inventory in jsonnet

Accessing the inventory from jsonnet compile types requires you to import `jsonnet/kapitan.libjsonnet`, which includes the native_callback functions glueing reclass to jsonnet (amongst others).

The jsonnet snippet below imports the inventory for the target you're compiling
and returns the java_opts for the elasticsearch data role:

```
local kap = import "lib/kapitan.libjsonnet";
inventory = kap.inventory();

{
    "data_java_opts": inventory.parameters.elasticsearch.roles.data.java_opts,
}
```

```
local kap = import "lib/kapitan.libjsonnet";
inventory = kap.inventory();

{
    "data_java_opts": inventory.parameters.elasticsearch.roles.data.java_opts,
}
```

#### Using the inventory in jinja2

Jinja2 types will pass the "inventory" and whatever target vars as context keys in your template.

This snippet renders the same java_opts for the elasticsearch data role:

```
java_opts for elasticsearch data role are: {{ inventory.parameters.elasticsearch.roles.data.java_opts }}
```


#### Jinja2 jsonnet templating

Such as reading the inventory within jsonnet, Kapitan also provides a function to render a Jinja2 template file. Again, importing "kapitan.jsonnet" is needed.

The jsonnet snippet renders the jinja2 template in templates/got.j2:

```
local kap = import "lib/kapitan.libjsonnet";

{
    "jon_snow": kap.jinja2_template("templates/got.j2", { is_dead: false }),
}
```

It's up to you to decide what the output is.


### kapitan inventory

Rendering the inventory for the _minikube-es_ target:

```
$ kapitan inventory -t minikube-es
...
classes:
  - cluster.minikube
  - app.elasticsearch
environment: base
parameters:
  cluster:
    name: minikube
    user: minikube
  elasticsearch:
    image: quay.io/pires/docker-elasticsearch-kubernetes:5.5.0
    java_opts: -Xms512m -Xmx512m
    masters: 1
    replicas: 1
    roles:
      client:
        image: quay.io/pires/docker-elasticsearch-kubernetes:5.5.0
        java_opts: -Xms512m -Xmx512m
        masters: 1
        replicas: 1
      data:
        image: quay.io/pires/docker-elasticsearch-kubernetes:5.5.0
        java_opts: -Xms512m -Xmx512m
        masters: 1
        replicas: 1
...
```

### kapitan searchvar





# Credits

* [Jsonnet](https://github.com/google/jsonnet)
* [Jinja2](http://jinja.pocoo.org/docs/2.9/)
* [reclass](https://github.com/madduck/reclass)



# Related projects

* [sublime-jsonnet-syntax](https://github.com/gburiola/sublime-jsonnet-syntax) - Jsonnet syntax highlighting for sublime text Edit

