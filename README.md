# logstash-eck-ingress-controller

This page will guide to configure logstash to work together with ECK and being acessible via ingress controller.

### Scenario

In this scenario we will assume you have read the (eck guide)[https://github.com/framsouza/eck] or e(ck saml setup)[https://github.com/framsouza/eck-saml-hot-warm-cold] in order to have an ECK & Kibana up and running.
Assuming you should have running the following resources:

- ECK Elasticsearch
- ECK Kibana
- Ingress Controller

All of them you can found at the repositories linked above.

Here we will deploy the following resouces:
- ECK beats
- Logstash deployment & services
- ConfigMap tcp service
- ConfigMap Logstash
