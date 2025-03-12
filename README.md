# Exporting Kubernetes Logs to OpenSearch Using Fluent-Bit on OCI

This tutorial focuses on OCI managed services (Kubernetes and OpenSearch),
however with some modifications may be applied to solutions that are hosted elsewhere.

The selection of Fluent-Bit has been dictated by its
lightweight nature among other log collection and processing tools (Fluentd, Logstash).

### OpenSearch configuration
Using OpenSearch Dashboards a role fluent-bit has been created allowing writing to the index 'k8s_logs'.
Internal user 'fluent-bit' has been then created and assigned the 'fluent-bit' role.
This users' credentials will be later used in fluent-bit configuration.

### Kubernetes configuration
The tutorial presumes having a deployed OCI k8s cluster with configured networking setup.

#### For accessing cluster create kubeconfig:
        oci ce cluster create-kubeconfig --cluster-id ocid1.cluster.oc1.$REGION.$ID --file $HOME/.kube/config --region $REGION --token-version 2.0.0 --kube-endpoint PRIVATE_ENDPOINT

#### Install fluent-bit daemonset
        helm repo add fluent https://fluent.github.io/helm-charts
        helm install fluent-bit fluent/fluent-bit

#### Validate installation
        helm list
        kubectl get daemonsets --all-namespaces

#### Edit fluent-bit configuration to specify sources and targets of the logs
        kubectl edit configmaps fluent-bit

### Fluent-bit configuration
Fluent-bit configuration has the following parts:

        [SERVICE]
            Daemon Off
            Flush 1
            Log_Level trace
            Log_File /var/log/fluent-bit.log
            Parsers_File /fluent-bit/etc/parsers.conf
            Parsers_File /fluent-bit/etc/conf/custom_parsers.conf
            HTTP_Server On
            HTTP_Listen 0.0.0.0
            HTTP_Port 2020
            Health_Check On

Setting logging verbosity level is optional
Allowed values are: off, error, warn, info, debug, and trace.
(https://docs.fluentbit.io/manual/administration/configuring-fluent-bit/classic-mode/configuration-file)

By deafult, logs are read from the kubelet systemd unit as well as from containers:

        [INPUT]
            Name tail
            Path /var/log/containers/*.log
            multiline.parser docker, cri
            Tag kube.*
            Mem_Buf_Limit 5MB
            Skip_Long_Lines On

        [INPUT]
            Name systemd
            Tag host.*
            Systemd_Filter _SYSTEMD_UNIT=kubelet.service
            Read_From_Tail On

The output sections describes target configuration:

        [OUTPUT]
            name               opensearch
            match              *
            host               aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa.opensearch.xx-yyyyyyyyy-1.oci.oraclecloud.com
            port               9200
            index              osd_managed_k8s_logs
            type               my_type
            HTTP_User          fluent-bit
            HTTP_Passwd        your_very_secure_password_MxSbAnSsvIHw4C9FmBAs3eG91eazyKIq0vt1MhGk1cj4UYJEHpSbAnSsvIHw4C9FmBAs3eG91eazKIq0
            tls                on
            tls.verify         on
            suppress_type_name on

### Node configuration
The nodes' OS (Oracle Linux Server) has Security Enhanced Linux (SELinux) kernel modifications, which
by default prevents fluent-bit from reading logs.

OCI > Developer Services > Kubernetes Clusters (OKE) > your_cluster >
> Node pools > your_pool > edit > Advanced options > Initialization script > Download Current Script

Edit the script and add these lines to allow fluent-bit access logs:

    sudo ausearch -c 'flb-pipeline' --raw | sudo audit2allow -M my-flbpipeline
    sudo semodule -X 300 -i my-flbpipeline.pp
    sudo ausearch -c 'flb-logger' --raw | sudo audit2allow -M my-flblogger
    sudo semodule -X 300 -i my-flblogger.pp
    sudo ausearch -c 'pmdaproc' --raw | sudo audit2allow -M my-pmdaproc
    sudo semodule -X 300 -i my-pmdaproc.pp

### Validating the setup
        sudo vim /var/log/containers/test.log
            ### this is a test log

OpenSearch Dashboards` query workbench:

          SELECT *
            FROM k8s_logs
           WHERE log LIKE '%this is a test log%'
        ORDER BY @timestamp DESC
           LIMIT 1
