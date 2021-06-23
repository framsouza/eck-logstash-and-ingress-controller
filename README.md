# logstash-eck-ingress-controller

This page will guide you to configure logstash to work together with ECK and being acessible via ingress controller.

In this scenario we will assume you have read the [eck guide](https://github.com/framsouza/eck) or [eck saml setup](https://github.com/framsouza/eck-saml-hot-warm-cold) in order to have an ECK & Kibana up and running.
I am Assuming you should have running the following resources:

- ECK Elasticsearch
- ECK Kibana
- Ingress Controller

All of them you can find at the repositories linked above.

Here we will deploy the following resouces:
- ECK beats
- Logstash deployment & services
- ConfigMap tcp service
- ConfigMap Logstash

## Architecture
 
In this scenario we are shipping Kubernetes logs using Filebeat (as part of the ECK) and sending it to a logstash deployment running on Kubernetes. Currently we do not support Logstash managed by ECK, and we need to deploy it as a deployment.

The flow is the quite easy: filebeat -> logstash -> elasticsearch <- kibana, all of this deployed on top of Kubernetes and being accesible via Ingress Controller.


## Logstash manifest explanation

Here I will highly the main configuration at the logstash manifest, you should adjust it to your use case

### Environmant variables

We need to define logstash output configuration, as usual we are send the events to Elasticsearch so here I am defining the Elasticsearch credentials. You should adjust the ES_HOSTS to your Elasticsearch service name and also adjust the ES_PASSWORD to point to the right secret name.

```
        env:
        - name: ES_HOSTS
          value: "https://elastic-prd-es-http:9200"
        - name: ES_USER
          value: "elastic"
        - name: ES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: elastic-prd-es-elastic-user
              key: elastic
```

With that we can refer use these variables into pipeline output configuration.

### Volumes
We are mounting 3 volumes into logstash: Configuration volume, Pipeline volume and Certs volume.
The configuration volume (_config-volume_) is used to store the logstash.yml and pipelines.yml. The volume called _volume-pipeline_ contains the pipeline itself, in this example we are using multiples pipeline so we have 2 pipelines defined. Then we have the certs volume which is used to connect to the Elasticsearch cluster.

The config-volume and volume-pipeline are mounted refering to a ConfigMap which contains the respective configuration. The cert-ca is mounted pointing to the Elasticsearch certs secret.

```
        volumeMounts:
          - name: config-volume
            mountPath: /usr/share/logstash/config
          - name: volume-pipeline
            mountPath: /usr/share/logstash/pipeline
          - name: cert-ca
            mountPath: "/etc/logstash/certificates"
      volumes:
      - name: config-volume
        configMap:
          name: logstash-multiple-pipeline
          items:
            - key: logstash.yml
              path: logstash.yml
            - key: pipelines.yml
              path: pipelines.yml
      - name: volume-pipeline
        configMap:
          name: logstash-multiple-pipeline
          items:
            - key: pipeline-00.config
              path: pipeline-00.config
            - key: pipeline-01.config
              path: pipeline-01.config
      - name: cert-ca
        secret:
          secretName: elastic-prd-es-http-certs-public
```

### Logstash service
The logstash service is quite simple, you can take a look at the [manifest](https://github.com/framsouza/logstash-eck-ingress-controller/blob/main/service-logstash.yml) and apply it without any change.

### Logstash configmap
Here we difine the configuration itself, I am using only one configMap to define the logstash.yml, pipelines.yml and pipelines definition.
Again, we are definiting a multiples pipeline configuration the pipeline-00.config and pipeline-01.config. You should adjust it to your use case with your filters and respective inputs. For this examples I am using beats input and Elasticsearch output.


```

apiVersion: v1
kind: ConfigMap
metadata:
  name: logstash-multiple-pipeline
data:
  logstash.yml: |-
    http.host: "0.0.0.0"
  pipelines.yml: |-
    - pipeline.id: pipeline00
      path.config: "/usr/share/logstash/pipeline/pipeline-00.config"
    - pipeline.id: pipeline01
      path.config: "/usr/share/logstash/pipeline/pipeline-01.config"
  pipeline-00.config: |-
    input { }
    filter { }
    output {
      stdout {
        codec => rubydebug
      }
    }

  pipeline-01.config: |-
    input {
      beats {
        port => 5044
      }
    }
    output {
      elasticsearch {
        index => "logstash-%{[@metadata][beat]}"
        hosts => [ "${ES_HOSTS}" ]
        user => "${ES_USER}"
        password => "${ES_PASSWORD}"
        cacert => '/etc/logstash/certificates/ca.crt'
      }
    }
```

Once you understand the manifest, you can apply the deployment-logstash.yml, service-logstash.yml and configmap-logstash.yml. Remember to first apply the configMap manifest otherwise the logstash will not be able to be created without it.

## Ingress Controller
Once you have the ingress controller up and running, we need to edit the ingress-nginx controller to be able to handle TCP requests. To do so you must add the following line into the deployment _--tcp-services-configmap=$(POD_NAMESPACE)/tcp-services_. It should looks like this:

```
      containers:
      - args:
        - /nginx-ingress-controller
        - --publish-service=$(POD_NAMESPACE)/ingress-nginx-controller
        - --election-id=ingress-controller-leader
        - --ingress-class=nginx
        - --configmap=$(POD_NAMESPACE)/ingress-nginx-controller
        - --validating-webhook=:8443
        - --validating-webhook-certificate=/usr/local/certificates/cert
        - --validating-webhook-key=/usr/local/certificates/key
        - --tcp-services-configmap=$(POD_NAMESPACE)/tcp-services
```

Once this is done, your ingress controller are almost ready to handle TCP connection, now you need to edit the ingress-nginx service and add the following lines:

```
  - name: proxied-tcp-5044
    nodePort: 30476
    port: 5044
    protocol: TCP
    targetPort: 5044
```

It will opened up the 5044 door for the service.
For this communication happens with logstash, you need to create an [extra configMap](https://github.com/framsouza/logstash-eck-ingress-controller/blob/main/configmap-tcp-services.yml) to refer to the logstash deployment. If you take a look at the flag you added at the ingress deployment (--tcp-services-configmap=$(POD_NAMESPACE)/tcp-service) this is pointing to an existing configMap where the key is the external port to use and the value indicates the service to expose.

```

apiVersion: v1
kind: ConfigMap
metadata:
  name: tcp-services
  namespace: ingress-nginx
data:
  5044: "default/logstash:5044"
```

Remember to create this configMap at the same namespace as the ingress deployment is running.

I have a domain called framsouza.co attached to this ingress controller. It means I can access my logstash using the following configuration tcp://framsouza.co:5044. You can run a telnet to the te connection, like this:

```
telnet framsouza.co 5044
Trying 35.205.61.152...
Connected to framsouza.co.
Escape character is '^]'.
```

## Filebeat

Now we have logstash deployment up and running together with the ingress changes, we can deploy the filebeat as part of the ECK, you can take at the [beats manifest](https://github.com/framsouza/logstash-eck-ingress-controller/blob/main/filebeat.yml) but this is very easy, the only configuration I changed was the output.

```
  config:
    filebeat.inputs:
    - type: container
      paths:
      - /var/log/containers/*.log
    output.logstash:
      hosts: ["framsouza.co:5044"]
```

I hope it helps you,

Cheers ;)
