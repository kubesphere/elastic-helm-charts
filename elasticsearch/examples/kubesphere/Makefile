default: test

PREFIX := elasticsearch
NAMESPACE ?= default
TIMEOUT := 1200s
PATCH := $(shell cat es-patch-torlence.json)
Tag := 7.11.1

# Install
# eg.  make install -e NAMESPACE=test
install:
	helm upgrade --install  $(PREFIX)-master ../../ --values master.yaml -n $(NAMESPACE) --set imageTag="$(Tag)"
	helm upgrade --install  $(PREFIX)-data ../../  --values data.yaml -n  $(NAMESPACE) --set imageTag="$(Tag)"

# Patch custom tolerences 
patch: 
	kubectl patch sts elasticsearch-master --patch  '$(PATCH)' -n  $(NAMESPACE) 
	kubectl patch sts elasticsearch-data --patch  '$(PATCH)' -n  $(NAMESPACE)

# Uninstall
# eg: make uninstall -e NAMESPACE=test
uninstall:
	helm del $(PREFIX)-master -n  $(NAMESPACE)
	helm del $(PREFIX)-data -n  $(NAMESPACE)
