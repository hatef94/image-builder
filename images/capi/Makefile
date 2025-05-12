# If you update this file, please follow
# https://suva.sh/posts/well-documented-makefiles

# Ensure Make is run with bash shell as some syntax below is bash-specific
SHELL := /usr/bin/env bash

.DEFAULT_GOAL := help

export PATH := $(PATH):$(CURDIR)/.local/bin
# Defining the canonical location of the Packer binary which will be used for
# `packer init`, `packer build` and `packer validate` commands.
PACKER=$(shell if [ $$(command -v packer | grep -v sbin) ]; then echo $$(command -v packer); else echo $(CURDIR)/.local/bin/packer; fi)

#export IB_VERSION ?= $(shell git describe --dirty)
export IB_VERSION ?= $(shell git describe --always)
## --------------------------------------
## Help
## --------------------------------------
##@ Helpers
help: ## Display this help
	@echo NOTE
	@echo '  The "build-node-ova" targets have analogue "clean-node-ova" targets for'
	@echo '  cleaning artifacts created from building OVAs using a local'
	@echo '  hypervisor.'
	@awk 'BEGIN {FS = ":.*##"; printf "\nUsage:\n  make \033[36m<target>\033[0m\n"} /^[a-zA-Z0-9_-]+:.*?##/ { printf "  \033[36m%-35s\033[0m %s\n", $$1, $$2 } /^##@/ { printf "\n\033[1m%s\033[0m\n", substr($$0, 5) } ' $(MAKEFILE_LIST)

.PHONY: version
version: ## Display version of image-builder
	@echo $(IB_VERSION)

GET_UBUNTU_DOTTED_SEMVER=$(strip \
	$(eval _UBUNTU_SEMVER:=$1) \
	$(eval _UBUNTU_SEMVER:=$(subst ubuntu-2204,ubuntu-22.04,$(_UBUNTU_SEMVER))) \
	$(eval _UBUNTU_SEMVER:=$(subst ubuntu-2404,ubuntu-24.04,$(_UBUNTU_SEMVER))) \
	$(_UBUNTU_SEMVER) \
)

## --------------------------------------
## Dependencies
## --------------------------------------
##@ Dependencies

.PHONY: deps
deps: ## Installs/checks all dependencies
deps: deps-ova deps-raw

.PHONY: deps-common
deps-common: ## Installs/checks dependencies common to most builds
deps-common:
	hack/ensure-python.sh
	hack/ensure-ansible.sh
	hack/ensure-packer.sh


.PHONY: deps-ova
deps-ova: ## Installs/checks dependencies for OVA builds
deps-ova: deps-common
	hack/ensure-ovftool.sh
	$(PACKER) init packer/config.pkr.hcl

.PHONY: deps-raw
deps-raw: ## Installs/checks dependencies for RAW builds
deps-raw: deps-common
	$(PACKER) init packer/config.pkr.hcl


.PHONY: deps-ignition
deps-ignition: ## Installs/checks dependencies for generating Ignition files
deps-ignition:
	hack/ensure-jq.sh
	hack/ensure-ct.sh

.PHONY: deps-release
deps-release: ## Installs/checks dependencies for project releases
	hack/ensure-go.sh
	hack/ensure-jq.sh
	hack/ensure-yq.sh

.PHONY: deps-lint
deps-lint: ## Installs/checks dependencies for image-builder code linting
deps-lint: deps-common
	hack/ensure-ansible-lint.sh
	hack/ensure-shellcheck.sh

## --------------------------------------
## Packer Configuration
## --------------------------------------


.PHONY: set-ssh-password
set-ssh-password:
	hack/set-ssh-password.sh

## --------------------------------------
## Packer flags
## --------------------------------------
# Set Packer color to true if not already set in env variables
# Only valid for builds
ifneq (,$(findstring build-, $(MAKECMDGOALS)))
# A build target
PACKER_COLOR ?= true
PACKER_FLAGS += -color=$(PACKER_COLOR)
endif

# If FOREGROUND=1 then Packer will set headless to false, causing local builds
# to build in the foreground, with a UI. This is very useful when debugging new
# platforms or issues with existing ones.
ifeq (1,$(strip $(FOREGROUND)))
PACKER_FLAGS += -var="headless=false"
endif

# If ON_ERROR_ASK=1 then Packer will set -on-error to ask, causing the Packer
# build to pause when any error happens, instead of simply exiting. This is
# useful when debugging unknown issues logging into the remote machine via ssh.
ifeq (1,$(strip $(ON_ERROR_ASK)))
PACKER_FLAGS += -on-error=ask
endif

# ssh_private_key_file and ssh_public_key are needed to pass ssh keypair
# from its host to the packer guest machine, so boot managers like ignition
# could make use of the key in its config.
# SSH_PRIVATE_KEY_FILE is name of the file that contains the private key.
# SSH_PUBLIC_KEY_FILE is name of the file that contains the public key.
ifneq (,$(strip $(SSH_PRIVATE_KEY_FILE)))
PACKER_FLAGS += -var ssh_private_key_file="$(SSH_PRIVATE_KEY_FILE)"
endif

ifneq (,$(strip $(SSH_PUBLIC_KEY_FILE)))
PACKER_FLAGS += -var ssh_public_key="$(shell cat ${SSH_PUBLIC_KEY_FILE})"
endif

# Since OpenSSH 9.0+ 'scp' uses SFTP protocol instead of legacy SCP protocol, which causes building errors like:
#
# bash: line 1: /usr/lib/sftp-server: No such file or directory\nscp: Connection closed\r\n""
#
# However, -O option is not available in older OpenSSH version, so we cannot always set it as an option to use.
# To provide better out-of-the-box experience for users with newer versions of OpenSSH, we conditionally ensure
# -O is used when used OpenSSH version requires it.
#
# See https://github.com/kubernetes-sigs/image-builder/issues/859 and
# https://github.com/hashicorp/packer-plugin-ansible/issues/100 for more details.
ifeq ($(shell test $$(ssh -V 2>&1 | cut -d _ -f2 | cut -d . -f1) -ge 9; echo $$?),0)
	# Use ?= to retain possible existing value of environment variable. If it is already declared, we assume user to be
	# aware of OpenSSH version they use and it is up to the user to specify "-O" option as well if needed.
  export ANSIBLE_SCP_EXTRA_ARGS ?= "-O"
endif

# If DEBUG=1 then Packer will set -debug, enabling debug mode for builds, providing
# more verbose logging
ifeq (1,$(strip $(DEBUG)))
PACKER_FLAGS += -debug
endif

# We want the var files passed to Packer to have a specific order, because the
# precedence of the variables they contain depends on the order. Files listed
# later on the CLI have higher precedence. We want the common var files found in
# packer/config to be listed first, then the var files that specific to the
# provider, then any user-supplied var files so that a user can override what
# they need to.

# A list of variable files given to Packer to configure things like the versions
# of Kubernetes, CNI, and ContainerD to install. Any additional files from the
# environment are appended.
COMMON_NODE_VAR_FILES := packer/config/kubernetes.json \
					packer/config/cni.json \
					packer/config/containerd.json \
					packer/config/wasm-shims.json \
					packer/config/ansible-args.json \
					packer/config/goss-args.json \
					packer/config/common.json \
					packer/config/additional_components.json

# Initialize a list of flags to pass to Packer. This includes any existing flags
# specified by PACKER_FLAGS, as well as prefixing the list with the variable
# files from COMMON_VAR_FILES, with each file prefixed by -var-file=.
#
# Any existing values from PACKER_FLAGS take precendence over variable files.
PACKER_NODE_FLAGS := $(foreach f,$(abspath $(COMMON_NODE_VAR_FILES)),-var-file="$(f)" ) \
				$(PACKER_FLAGS)
ABSOLUTE_PACKER_VAR_FILES := $(foreach f,$(abspath $(PACKER_VAR_FILES)),-var-file="$(f)" )

## --------------------------------------
## Platform and version combinations
## --------------------------------------
FLATCAR_VERSIONS		:=	flatcar
PHOTON_VERSIONS			:=  photon-4 photon-5
UBUNTU_VERSIONS			:=	ubuntu-2204 ubuntu-2204-efi ubuntu-2404 ubuntu-2404-efi

# Set Flatcar Container Linux channel and version if not supplied
FLATCAR_CHANNEL ?= stable
FLATCAR_VERSION ?= current
ifeq ($(FLATCAR_VERSION),current)
override FLATCAR_VERSION := $(shell hack/image-grok-latest-flatcar-version.sh $(FLATCAR_CHANNEL))
endif

export FLATCAR_CHANNEL FLATCAR_VERSION

PLATFORMS_AND_VERSIONS	:=	$(PHOTON_VERSIONS) \
							$(UBUNTU_VERSIONS) \
							$(FLATCAR_VERSIONS)

NODE_OVA_LOCAL_BUILD_NAMES			:=	$(addprefix node-ova-local-,$(PLATFORMS_AND_VERSIONS))
NODE_OVA_LOCAL_VMX_BUILD_NAMES		:=	$(addprefix node-ova-local-vmx-,$(PLATFORMS_AND_VERSIONS))
NODE_OVA_LOCAL_BASE_BUILD_NAMES		:=	$(addprefix node-ova-local-base-,$(PLATFORMS_AND_VERSIONS))
NODE_OVA_VSPHERE_BUILD_NAMES		:=	$(addprefix node-ova-vsphere-,$(PLATFORMS_AND_VERSIONS))
NODE_OVA_VSPHERE_BASE_BUILD_NAMES		:=	$(addprefix node-ova-vsphere-base-,$(PLATFORMS_AND_VERSIONS))
NODE_OVA_VSPHERE_CLONE_BUILD_NAMES		:=	$(addprefix node-ova-vsphere-clone-,$(PLATFORMS_AND_VERSIONS))
## --------------------------------------
## Dynamic build targets
## --------------------------------------
NODE_OVA_LOCAL_BUILD_TARGETS	:= $(addprefix build-,$(NODE_OVA_LOCAL_BUILD_NAMES))
NODE_OVA_LOCAL_VMX_BUILD_TARGETS	:= $(addprefix build-,$(NODE_OVA_LOCAL_VMX_BUILD_NAMES))
NODE_OVA_LOCAL_BASE_BUILD_TARGETS	:= $(addprefix build-,$(NODE_OVA_LOCAL_BASE_BUILD_NAMES))
NODE_OVA_LOCAL_VALIDATE_TARGETS	:= $(addprefix validate-,$(NODE_OVA_LOCAL_BUILD_NAMES))
NODE_OVA_VSPHERE_BUILD_TARGETS	:= $(addprefix build-,$(NODE_OVA_VSPHERE_BUILD_NAMES))
NODE_OVA_VSPHERE_BASE_BUILD_TARGETS	:= $(addprefix build-,$(NODE_OVA_VSPHERE_BASE_BUILD_NAMES))
NODE_OVA_VSPHERE_CLONE_BUILD_TARGETS	:= $(addprefix build-,$(NODE_OVA_VSPHERE_CLONE_BUILD_NAMES))
RAW_BUILD_TARGETS      := $(addprefix build-,$(RAW_BUILD_NAMES))
RAW_VALIDATE_TARGETS   := $(addprefix validate-,$(RAW_BUILD_NAMES))

.PHONY: $(NODE_OVA_LOCAL_BUILD_TARGETS)
$(NODE_OVA_LOCAL_BUILD_TARGETS): deps-ova set-ssh-password
	$(PACKER) build $(PACKER_NODE_FLAGS) -var-file="packer/ova/packer-common.json" -var-file="$(abspath packer/ova/$(subst build-node-ova-local-,,$@).json)" -except=vsphere -only=vmware-iso $(ABSOLUTE_PACKER_VAR_FILES) packer/ova/packer-node.json

.PHONY: $(NODE_OVA_LOCAL_VALIDATE_TARGETS)
$(NODE_OVA_LOCAL_VALIDATE_TARGETS): deps-ova set-ssh-password
	$(PACKER) validate $(PACKER_NODE_FLAGS) -var-file="packer/ova/packer-common.json" -var-file="$(abspath packer/ova/$(subst validate-node-ova-local-,,$@).json)" -except=vsphere -only=vmware-iso $(ABSOLUTE_PACKER_VAR_FILES) packer/ova/packer-node.json

.PHONY: $(NODE_OVA_LOCAL_VMX_BUILD_TARGETS)
$(NODE_OVA_LOCAL_VMX_BUILD_TARGETS): deps-ova set-ssh-password
	$(PACKER) build $(PACKER_NODE_FLAGS) -var-file="packer/ova/packer-common.json" -var-file="$(abspath packer/ova/$(subst build-node-ova-local-vmx-,,$@).json)" -var-file="packer/ova/vmx.json" -except=vsphere -except=vmware-iso -only=vmware-vmx $(ABSOLUTE_PACKER_VAR_FILES) packer/ova/packer-node.json

.PHONY: $(NODE_OVA_LOCAL_BASE_BUILD_TARGETS)
$(NODE_OVA_LOCAL_BASE_BUILD_TARGETS): deps-ova set-ssh-password
	$(PACKER) build $(PACKER_NODE_FLAGS) -var-file="packer/ova/packer-common.json" -var-file="$(abspath packer/ova/$(subst build-node-ova-local-base-,,$@).json)"  -except=vsphere -except=vmware-iso -except=vmware-vmx -only=vmware-iso-base $(ABSOLUTE_PACKER_VAR_FILES) packer/ova/packer-node.json

.PHONY: $(NODE_OVA_VSPHERE_BUILD_TARGETS)
$(NODE_OVA_VSPHERE_BUILD_TARGETS): deps-ova set-ssh-password
    # This uses a packer file builder to input unattend variables into a JSON file to be consumed by the python script before running the vsphere provisioner
	$(PACKER) build $(PACKER_NODE_FLAGS)  -var-file="packer/ova/packer-common.json" -var-file="$(abspath packer/ova/$(subst build-node-ova-vsphere-,,$@).json)" -var-file="packer/ova/vsphere.json"  -except=local -only=vsphere-iso $(ABSOLUTE_PACKER_VAR_FILES) -only=vsphere packer/ova/packer-node.json

.PHONY: $(NODE_OVA_VSPHERE_BASE_BUILD_TARGETS)
$(NODE_OVA_VSPHERE_BASE_BUILD_TARGETS): deps-ova set-ssh-password
	$(PACKER) build $(PACKER_NODE_FLAGS) -var-file="packer/ova/packer-common.json" -var-file="$(abspath packer/ova/$(subst build-node-ova-vsphere-base-,,$@).json)" -var-file="packer/ova/vsphere.json" -except=local -except=manifest -except=vsphere -only=vsphere-iso-base $(ABSOLUTE_PACKER_VAR_FILES) packer/ova/packer-node.json

.PHONY: $(NODE_OVA_VSPHERE_CLONE_BUILD_TARGETS)
$(NODE_OVA_VSPHERE_CLONE_BUILD_TARGETS): deps-ova set-ssh-password
	$(PACKER) build $(PACKER_NODE_FLAGS) -var-file="packer/ova/packer-common.json" -var-file="$(abspath packer/ova/$(subst build-node-ova-vsphere-clone-,,$@).json)" -var-file="packer/ova/vsphere.json" -except=local -only=vsphere-clone $(ABSOLUTE_PACKER_VAR_FILES) packer/ova/packer-node.json

.PHONY: $(RAW_BUILD_TARGETS)
$(RAW_BUILD_TARGETS): deps-raw set-ssh-password
	$(PACKER) build $(PACKER_NODE_FLAGS) -var-file="$(abspath packer/raw/$(subst build-,,$@).json)" $(ABSOLUTE_PACKER_VAR_FILES) packer/raw/packer.json

.PHONY: $(RAW_VALIDATE_TARGETS)
$(RAW_VALIDATE_TARGETS): deps-raw set-ssh-password
	$(PACKER) validate $(PACKER_NODE_FLAGS) -var-file="$(abspath packer/raw/$(subst validate-,,$@).json)" $(ABSOLUTE_PACKER_VAR_FILES) packer/raw/packer.json
## --------------------------------------
## Dynamic clean targets
## --------------------------------------
NODE_OVA_LOCAL_CLEAN_TARGETS := $(subst build-,clean-,$(NODE_OVA_LOCAL_BUILD_TARGETS))
.PHONY: $(NODE_OVA_LOCAL_CLEAN_TARGETS)
$(NODE_OVA_LOCAL_CLEAN_TARGETS):
	rm -fr output/$(subst clean-node-ova-local-,,$@)-kube*

QEMU_CLEAN_TARGETS := $(subst build-,clean-,$(QEMU_BUILD_TARGETS))
.PHONY: $(QEMU_CLEAN_TARGETS)
$(QEMU_CLEAN_TARGETS):
	rm -fr output/$(subst clean-qemu-,,$@)-kube*

RAW_CLEAN_TARGETS := $(subst build-,clean-,$(RAW_BUILD_TARGETS))
.PHONY: $(RAW_CLEAN_TARGETS)
$(RAW_CLEAN_TARGETS):
	rm -fr output/$(subst clean-raw-,,$@)-kube*

## --------------------------------------
## Document dynamic build targets
## --------------------------------------
##@ Builds
build-node-ova-local-flatcar: ## Builds Flatcar stable Node OVA w local hypervisor
build-node-ova-local-photon-4: ## Builds Photon 4 Node OVA w local hypervisor
build-node-ova-local-photon-5: ## Builds Photon 5 Node OVA w local hypervisor
build-node-ova-local-all: $(NODE_OVA_LOCAL_BUILD_TARGETS) ## Builds all Node OVAs w local hypervisor

build-node-ova-vsphere-flatcar: ## Builds Flatcar stable Node OVA and template on vSphere
build-node-ova-vsphere-photon-4: ## Builds Photon 4 Node OVA and template on vSphere
build-node-ova-vsphere-photon-5: ## Builds Photon 5 Node OVA and template on vSphere
build-node-ova-vsphere-ubuntu-2204: ## Builds Ubuntu 22.04 Node OVA and template on vSphere
build-node-ova-vsphere-ubuntu-2204-efi: ## Builds Ubuntu 22.04 Node OVA and template on vSphere that EFI boots
build-node-ova-vsphere-ubuntu-2404: ## Builds Ubuntu 24.04 Node OVA and template on vSphere
build-node-ova-vsphere-ubuntu-2404-efi: ## Builds Ubuntu 24.04 Node OVA and template on vSphere that EFI boots
build-node-ova-vsphere-all: $(NODE_OVA_VSPHERE_BUILD_TARGETS) ## Builds all Node OVAs and templates on vSphere

build-node-ova-vsphere-clone-photon-4: ## Builds Photon 4 Node OVA and template on vSphere
build-node-ova-vsphere-clone-photon-5: ## Builds Photon 5 Node OVA and template on vSphere
build-node-ova-vsphere-clone-ubuntu-2204: ## Builds Ubuntu 22.04 Node OVA and template on vSphere
build-node-ova-vsphere-clone-ubuntu-2204-efi: ## ## Builds Ubuntu 22.04 Node OVA and template on vSphere that EFI boots
build-node-ova-vsphere-clone-ubuntu-2404: ## Builds Ubuntu 24.04 Node OVA and template on vSphere
build-node-ova-vsphere-clone-ubuntu-2404-efi: ## ## Builds Ubuntu 24.04 Node OVA and template on vSphere that EFI boots
build-node-ova-vsphere-clone-all: $(NODE_OVA_VSPHERE_CLONE_BUILD_TARGETS) ## Builds all Node OVAs and templates on vSphere

build-node-ova-vsphere-base-photon-4: ## Builds base Photon 4 Node OVA and template on vSphere
build-node-ova-vsphere-base-photon-5: ## Builds base Photon 5 Node OVA and template on vSphere
build-node-ova-vsphere-base-ubuntu-2204: ## Builds base Ubuntu 22.04 Node OVA and template on vSphere
build-node-ova-vsphere-base-ubuntu-2204-efi: ## Builds Ubuntu 22.04 Node OVA and template on vSphere that EFI boots
build-node-ova-vsphere-base-ubuntu-2404: ## Builds base Ubuntu 24.04 Node OVA and template on vSphere
build-node-ova-vsphere-base-ubuntu-2404-efi: ## Builds Ubuntu 24.04 Node OVA and template on vSphere that EFI boots
build-node-ova-vsphere-base-all: $(NODE_OVA_VSPHERE_BASE_BUILD_TARGETS) ## Builds all base Node OVAs and templates on vSphere

build-node-ova-local-vmx-photon-4: ## Builds Photon 4 Node OVA from VMX file w local hypervisor
build-node-ova-local-vmx-photon-5: ## Builds Photon 5 Node OVA from VMX file w local hypervisor

build-node-ova-local-base-photon-4: ## Builds Photon 4 Base Node OVA w local hypervisor
build-node-ova-local-base-photon-5: ## Builds Photon 5 Base Node OVA w local hypervisor

build-raw-flatcar: ## Builds Flatcar RAW image
build-raw-ubuntu-2204: ## Builds Ubuntu 22.04 RAW image
build-raw-ubuntu-2204-efi: ## Builds Ubuntu 22.04 RAW image that EFI boots
build-raw-all: $(RAW_BUILD_TARGETS) ## Builds all RAW images

## --------------------------------------
## Document dynamic validate targets
## --------------------------------------
##@ Validate packer config
validate-node-ova-local-flatcar: ## Validates Flatcar stable Node OVA Packer config w local hypervisor
validate-node-ova-local-photon-4: ## Validates Photon 4 Node OVA Packer config w local hypervisor
validate-node-ova-local-photon-5: ## Validates Photon 5 Node OVA Packer config w local hypervisor
validate-node-ova-local-ubuntu-2204: ## Validates Ubuntu 22.04 Node OVA Packer config w local hypervisor
validate-node-ova-local-ubuntu-2404: ## Validates Ubuntu 24.04 Node OVA Packer config w local hypervisor
validate-node-ova-local-all: $(NODE_OVA_LOCAL_VALIDATE_TARGETS) ## Validates all Node OVAs Packer config w local hypervisor

validate-node-ova-local-vmx-photon-4: ## Validates Photon 4 Node OVA from VMX file w local hypervisor
validate-node-ova-local-vmx-photon-5: ## Validates Photon 5 Node OVA from VMX file w local hypervisor
validate-node-ova-local-vmx-ubuntu-2204: ## Validates Ubuntu 22.04 Node OVA from VMX file w local hypervisor
validate-node-ova-local-vmx-ubuntu-2404: ## Validates Ubuntu 24.04 Node OVA from VMX file w local hypervisor

validate-node-ova-local-base-photon-4: ## Validates Photon 4 Base Node OVA w local hypervisor
validate-node-ova-local-base-photon-5: ## Validates Photon 5 Base Node OVA w local hypervisor
validate-node-ova-local-base-ubuntu-2204: ## Validates Ubuntu 22.04 Base Node OVA w local hypervisor
validate-node-ova-local-base-ubuntu-2404: ## Validates Ubuntu 24.04 Base Node OVA w local hypervisor

validate-raw-flatcar: ## Validates Flatcar RAW image packer config
validate-raw-ubuntu-2204: ## Validates Ubuntu 22.04 RAW image packer config
validate-raw-ubuntu-2204-efi: ## Validates Ubuntu 22.04 RAW EFI image packer config
validate-raw-all: $(RAW_VALIDATE_TARGETS) ## Validates all RAW Packer config

validate-all: validate-node-ova-local-all \
	validate-raw-all

validate-all: ## Validates the Packer config for all build targets

.PHONY: lint
lint: ## Runs linters on image-builder code
sh_files = $(shell find . -type f -name "*.sh")
lint: deps-lint
	ansible-lint ansible/
# ignore error code since shellcheck exits with Error 1 if problems are found despite running properly
	-@for f in $(sh_files); do (shellcheck -x $$f); done

.PHONY: lint-fix
lint-fix: ## Runs linters on image-builder code and fixes issues
lint-fix: deps-lint
	ansible-lint --fix=all ansible/

.PHONY: lint-ignore
lint-ignore: ## Runs linters on image-builder code and creates an "ignore" file
lint-ignore: deps-lint
	ansible-lint --generate-ignore ansible/

## --------------------------------------
## Clean targets
## --------------------------------------
##@ Cleaning
.PHONY: clean
clean: ## Removes all image output directories and packer image cache
clean: $(NODE_OVA_LOCAL_CLEAN_TARGETS) $(QEMU_CLEAN_TARGETS) clean-packer-cache

.PHONY: clean-ova
clean-ova: ## Removes all ova image output directories (see NOTE at top of help)
clean-ova: $(NODE_OVA_LOCAL_CLEAN_TARGETS)

.PHONY: clean-raw
clean-raw: ## Removes all raw image output directories (see NOTE at top of help)
clean-raw: $(RAW_CLEAN_TARGETS)

.PHONY: clean-packer-cache
clean-packer-cache: ## Removes the packer cache
clean-packer-cache:
	rm -fr packer_cache/*
## --------------------------------------
## Sort JSON
## --------------------------------------
##@ Sort JSON

.PHONY: json-sort
json_files = $(shell find . -type f -name "*.json" | sort -u)
json-sort: ## Sort all JSON files alphabetically
	@for f in $(json_files); do (cat "$$f" | jq -S '.' >> "$$f".sorted && mv "$$f".sorted "$$f") || exit 1 ; done


## --------------------------------------
## Ignition
## --------------------------------------
##@ Ignition
.PHONY: gen-ignition
ignition_files = bootstrap-pass-auth bootstrap-cloud
gen-ignition: deps-ignition ## Generates Ignition files from CLC
	for f in $(ignition_files); do (ct < packer/files/flatcar/clc/$$f.yaml | jq '.' > packer/files/flatcar/ignition/$$f.json) || exit 1; done

## --------------------------------------
## ISO checksum updates
## --------------------------------------
##@ ISO checksum updates
.PHONY: update-all-iso-checksums
update-all-iso-checksums: update-ubuntu-iso-checksums \
	update-centos-9-iso-checksums \
	update-photon-iso-checksums \

update-all-iso-checksums: ## Updates checksums for all ISOs

.PHONY: update-ubuntu-iso-checksums
update-ubuntu-iso-checksums: ## Updates checksums for Ubuntu ISOs
	hack/update-iso-checksums.sh ubuntu false SHA256SUMS "*iso_file_name" 1
	$(MAKE) json-sort

.PHONY: update-centos-9-iso-checksums
update-centos-9-iso-checksums: ## Updates checksums for Centos 9 ISOs
	hack/update-iso-checksums.sh centos-9 false SHA256SUM "SHA256 (iso_file_name)" 4
	$(MAKE) json-sort

.PHONY: update-photon-iso-checksums
update-photon-iso-checksums: ## Updates checksums for Photon ISOs
	hack/update-iso-checksums.sh photon true
	$(MAKE) json-sort