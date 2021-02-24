default: test

include ../../../helpers/examples.mk

PREFIX := elasticsearch
NAMESPACE := default
TIMEOUT := 1200s

#install:
#	helm install  $(PREFIX)-master ../../ --values master.yaml -n $(NAMESPACE)
#	helm install  $(PREFIX)-data ../../  --values data.yaml -n  $(NAMESPACE)
#	helm upgrade --wait --timeout=$(TIMEOUT) --install --values client.yaml $(PREFIX)-client ../../

install-master:
	helm upgrade --install  $(PREFIX)-master ../../ --values master.yaml -n $(NAMESPACE) --set imageTag="7.10.2"

install-data:
	helm upgrade --install  $(PREFIX)-data ../../  --values data.yaml -n  $(NAMESPACE) --set imageTag="7.10.2"

test: install goss

purge:
	helm del $(PREFIX)-master -n  $(NAMESPACE)
	helm del $(PREFIX)-data -n  $(NAMESPACE)
#	helm del $(PREFIX)-client
