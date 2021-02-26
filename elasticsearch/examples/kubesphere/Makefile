default: test

include ../../../helpers/examples.mk

PREFIX := elasticsearch
RELEASE := elasticsearch-data
NAMESPACE ?= default
TIMEOUT := 1200s
PATCH := $(shell cat es-patch-torlence.json)
Tag := 7.11.1

test: install patch goss

install:
	helm upgrade --install  $(PREFIX)-master ../../ --values master.yaml -n $(NAMESPACE) --set imageTag="$(Tag)"
	helm upgrade --install  $(PREFIX)-data ../../  --values data.yaml -n  $(NAMESPACE) --set imageTag="$(Tag)"


patch: 
	kubectl patch sts elasticsearch-master --patch  '$(PATCH)' -n  $(NAMESPACE) 
	kubectl patch sts elasticsearch-data --patch  '$(PATCH)' -n  $(NAMESPACE)

uninstall:
	helm del $(PREFIX)-master -n  $(NAMESPACE)
	helm del $(PREFIX)-data -n  $(NAMESPACE)
