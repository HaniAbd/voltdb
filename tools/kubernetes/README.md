[comment]: # (This file is part of VoltDB.)
[comment]: # (Copyright © 2008-2018 VoltDB Inc.)

# Running a VoltDB Database Using Kubernetes

The accompanying scripts provide support for running VoltDB in a Kubernetes environment.
Use these scripts as templates, customizing them as described in the following sections
to match your requirements and/or environment.

## Setting Up the Deployment

Here's the procedure to setup a k8s deployment of VoltDB:

1. Download and untar the voltdb release kit.

2. cd kubernetes folder (this folder). In this folder you will find:

    build_image.sh                  Script to deploy voltdb clusters
    config_template.cfg             Deployment Parameter template (requires customization)
    voltdb-statefulset.yaml         k8s deployment spec (template)
    
    

3. The script build_image.sh along with an accompanying parameter file will enable you to deploy a VoltDB Cluster
   in k8s. The script has the following functions:
   
    build_image.sh <cluster-parameter-file> <option>
    
    <cluster-parameter-file> has the key-value pairs for the cluster settings
    <options> one or more of the following options.

    See the description of the individual functions for details
    
    -B, --build-voltdb-image        Builds the VoltDB docker image using docker build
    
    -M, --install-configmap         Installs the configmap for VoltDB init using kubectl create configmap
    
    -C, --configure-voltdb          Generates a statefulset.yaml to deploy the VoltDB cluster using
                                    voltdb-statefulset.yaml as a master template
    
    -S, --start-voltdb              Deploys the VoltDB cluster and starts the nodes using kubectl scale/create
    
    -P, --stop-voltdb               Gracefully shutdown VoltDB using kubectl exec voltadmin shutdown
    
    -D, --purge-persistent-claims   Purge persistent volume (PERSISTENT DATA WILL BE LOST)
                                    using kubectl delete pv ...
    
    -F, --force-voltdb-set          Terminate VoltDB statefulset ungracefully (POSSIBLE DATA LOSS)
                                    using kubectl delete pods and statefulsets for the cluster
    
    If you want to more than one option in an invocation, specify them separately and in the order
    in which you want them to execute. ie. "-B -M -C -S"



4. Make a new CONFIG_FILE by copying config_template.cfg.

    The filename that you choose for your template will be used as the Cluster Name for the voltdb cluster, 
    all outputs will be identified by this name.
   
    1. Set REP to the docker repository to push the image to (optional)
    2. Set IMAGE_TAG to the image tag (optional)
    3. Set NODECOUNT to the number of voltdb nodes (required)
    4. Set MEMORYSIZE to the required memory size to request from k8s for each node (default 4Gi)
    5. Set CPU_COUNT to the required number of cpus to request from k8s for each node (default 1)
    6. Set PVOLUME_SIZE to the size of the persistent volume needed for each node (default 1Gi)
    7. Set LICENSE_FILE to the location of your voltdb license file (default from the voltdb kit)
    8. Set DEPLOYMENT_FILE to the location of your VoltDB deployment file (required)
    9. Set SCHEMA_FILE to the location of your database startup schema (optional)
   10. Set CLASSES_JAR to the location of your database startup classes (options)
   11. Set EXTENSION_DIR, BUNDLES_DIR, LOG4J_FILE  to the location of your lib-extension, bundes, or logging properties (optional)

    Other customizations to consider:

    To set the Java Heap Size for VoltDB, edit the statefulset yaml file and set the following into the environment (env:) section:
          env:
            ...
            - name: MAX_HEAP_SIZE
              value: <required-heap-size> ex. 2G
            ...
     See VoltDB documentation for more information.

    Passing startup parameters to VoltDB:

    You can pass parameters to voltdb at startup by setting VOLTDB_START_ARGS in the k8s deployment yaml:
        env:
            ...
            - name: VOLTDB_START_ARGS
              value: "--missing=1 ..."
            ...
    See VoltDB documentation for more information.

    For your deployment there may be other things to configure such as persistent volume properties, sizes, etc.,
    you'll need to edit the voltdb-statefulset.yaml template and set your specific requirements there, and check
    it into your source control.

5. Build the image and voltdb-statefulset deployment file:

    This step creates the voltdb image and configures a kubernetes deployment file.
    The resulting deployment file will be named... The container image is customized
    with provided assets, and a initial database root is created with your settings.
    This root will be copied to persistent storage on first run of the database (node).

        ./build_image.sh CONFIG_FILE

## Starting and Stopping the Cluster

* To start the cluster:

        kubectl create -f K8S_DEPLOYMENT

* To stop the cluster, retaining the persistent volume(s):

        voltadmin pause --wait
        kubectl delete -f K8S_DEPLOYMENT

* To display persistent volume(s) and claim(s):

        kubectl get pvc
        kubectl get pv

* To delete volumes (all data in the database will be lost):

        kubectl delete <pv/pvc name>

    nb. next time the database is brought up it will be in the initial state

## Other Useful Commands

Here are some other commands that are useful (assuming the name of the statefulset is "voltdb" and its pods are voltdb-0, ...)

* To connect sqlcmd to a running cluster

        kubectl exec -it voltdb-0 -- sqlcmd --servers=localhost

* To proxy VoltDB ports to localhost ports (ports are remote[:local])

        kubectl port-forward statefulset/<cluster name> 8080 21212 21211

   You can then run voltdb commands locally, for example:

        $VOLTDB_HOME/bin/sqlcmd --servers=localhost --port=21212
        $VOLTDB_HOME/bin/voltadmin ...
