BASE_DIRECTORY=$(shell git rev-parse --show-toplevel)
RELEASE_BRANCH?=1-18
RELEASE?=$(shell cat $(BASE_DIRECTORY)/release/$(RELEASE_BRANCH)/RELEASE)
ARTIFACT_BUCKET?=my-s3-bucket
GIT_TAG?=$(shell cat ./$(RELEASE_BRANCH)/GIT_TAG)

REPO=kubernetes
COMPONENT=kubernetes/$(REPO)
CLONE_URL=https://github.com/$(COMPONENT).git

AWS_ACCOUNT_ID?=$(shell aws sts get-caller-identity --query Account --output text)
AWS_REGION?=us-west-2

IMAGE_REPO?=$(AWS_ACCOUNT_ID).dkr.ecr.$(AWS_REGION).amazonaws.com
IMAGE_REPO_PREFIX=kubernetes
IMAGE_TAG?=$(GIT_TAG)-eks-${RELEASE_BRANCH}-${RELEASE}

KUBE_BASE_REPO?=$(IMAGE_REPO)
GO_RUNNER_IMAGE_NAME?=kubernetes/go-runner
KUBE_PROXY_BASE_IMAGE_NAME?=kubernetes/kube-proxy-base
KUBE_BASE_GIT_TAG?=$(shell cat $(BASE_DIRECTORY)/projects/kubernetes/release/GIT_TAG)
KUBE_BASE_TAG?=${KUBE_BASE_GIT_TAG}-eks-${RELEASE_BRANCH}-${RELEASE}
GO_RUNNER_IMAGE?=$(KUBE_BASE_REPO)/$(GO_RUNNER_IMAGE_NAME):$(KUBE_BASE_TAG)
KUBE_PROXY_BASE_IMAGE?=$(KUBE_BASE_REPO)/$(KUBE_PROXY_BASE_IMAGE_NAME):$(KUBE_BASE_TAG)

# Since the image won't be available for the local-images target,
# need to override based on an image that is already available.
# We want to a better way than this to avoid having to change often.
PRESUBMIT_KUBE_BASE_TAG?=v0.4.2-ea45689a0da457711b15fa1245338cd0b636ad4b
PRESUBMIT_GO_RUNNER_IMAGE?=$(KUBE_BASE_REPO)/$(GO_RUNNER_IMAGE_NAME):$(PRESUBMIT_KUBE_BASE_TAG)
PRESUBMIT_KUBE_PROXY_BASE_IMAGE?=$(KUBE_BASE_REPO)/$(KUBE_PROXY_BASE_IMAGE_NAME):$(PRESUBMIT_KUBE_BASE_TAG)

PAUSE_SRC_DIR?=kubernetes/build/pause
PAUSE_DST_DIR?=_output/pause
PAUSE_IMAGE_NAME?=pause
PAUSE_IMAGE_TAG?=$(IMAGE_TAG)
PAUSE_IMAGE?=$(IMAGE_REPO)/$(IMAGE_REPO_PREFIX)/$(PAUSE_IMAGE_NAME):$(PAUSE_IMAGE_TAG)

.PHONY: update-version
update-version: binaries
	build/create_version_file.sh $(GIT_TAG) $(RELEASE_BRANCH)

.PHONY: update-checksums
update-checksums: binaries
	build/update_checksums.sh $(RELEASE_BRANCH)

.PHONY: binaries
binaries:
	build/create_binaries.sh $(CLONE_URL) $(RELEASE_BRANCH) $(GIT_TAG)

.PHONY: tarballs
tarballs:
	build/create_tarballs.sh $(RELEASE_BRANCH)

$(PAUSE_DST_DIR)/pause.c:
	mkdir -p $(PAUSE_DST_DIR)
	cp $(PAUSE_SRC_DIR)/linux/pause.c $(PAUSE_DST_DIR) || cp $(PAUSE_SRC_DIR)/pause.c $(PAUSE_DST_DIR)

pause: $(PAUSE_DST_DIR)/pause.c

.PHONY: local-images
local-images:
	build/create_images.sh $(RELEASE_BRANCH) $(PRESUBMIT_GO_RUNNER_IMAGE) $(PRESUBMIT_KUBE_PROXY_BASE_IMAGE) $(IMAGE_REPO) $(IMAGE_REPO_PREFIX) $(IMAGE_TAG) false
	buildctl \
		build \
		--frontend dockerfile.v0 \
		--opt platform=linux/amd64 \
		--local dockerfile=./docker/pause/ \
		--local context=./_output/pause \
		--opt build-arg:BASE_IMAGE=$(PRESUBMIT_GO_RUNNER_IMAGE) \
		--opt build-arg:VERSION=$(IMAGE_TAG) \
		--output type=oci,oci-mediatypes=true,name=$(PAUSE_IMAGE),dest=_output/$(RELEASE_BRANCH)/bin/linux/amd64/pause.tar

.PHONY: images
images:
	build/create_images.sh $(RELEASE_BRANCH) $(GO_RUNNER_IMAGE) $(KUBE_PROXY_BASE_IMAGE) $(IMAGE_REPO) $(IMAGE_REPO_PREFIX) $(IMAGE_TAG) true
	buildctl \
		build \
		--frontend dockerfile.v0 \
		--opt platform=linux/amd64,linux/arm64  \
		--local dockerfile=./docker/pause/ \
		--local context=./_output/pause \
		--opt build-arg:BASE_IMAGE=$(GO_RUNNER_IMAGE) \
		--opt build-arg:VERSION=$(IMAGE_TAG) \
		--output type=image,oci-mediatypes=true,\"name=$(PAUSE_IMAGE),$(IMAGE_REPO)/$(IMAGE_REPO_PREFIX)/$(PAUSE_IMAGE_NAME):3.2\",push=true

.PHONY: docker
docker: binaries pause
	build/create_docker_images.sh $(RELEASE_BRANCH) $(GO_RUNNER_IMAGE) $(KUBE_PROXY_BASE_IMAGE) $(IMAGE_REPO) $(IMAGE_REPO_PREFIX) $(IMAGE_TAG)
	docker \
		build\
		--build-arg BASE_IMAGE=$(GO_RUNNER_IMAGE) \
		--build-arg VERSION=$(IMAGE_TAG) \
		-t $(PAUSE_IMAGE) \
		-t $(IMAGE_REPO)/$(IMAGE_REPO_PREFIX)/$(PAUSE_IMAGE_NAME):3.2 \
		-f ./docker/pause/Dockerfile ./_output/pause/

.PHONY: docker-push
docker-push:
	build/docker_push.sh $(RELEASE_BRANCH) $(GO_RUNNER_IMAGE) $(KUBE_PROXY_BASE_IMAGE) $(IMAGE_REPO) $(IMAGE_REPO_PREFIX) $(IMAGE_TAG)
	docker push $(PAUSE_IMAGE)
	docker push $(IMAGE_REPO)/$(IMAGE_REPO_PREFIX)/$(PAUSE_IMAGE_NAME):3.2

.PHONY: checksums
checksums:
	build/create_release_checksums.sh $(RELEASE_BRANCH)

.PHONY: build
build: test pause tarballs clean-repo local-images checksums

.PHONY: release
release: binaries pause tarballs clean-repo local-images images checksums
	$(BASE_DIRECTORY)/release/copy_artifacts.sh $(COMPONENT) $(RELEASE_BRANCH) $(RELEASE)
	$(BASE_DIRECTORY)/release/s3_sync.sh $(RELEASE_BRANCH) $(RELEASE) $(ARTIFACT_BUCKET) $(REPO)
	echo "Done $(COMPONENT)"

.PHONY: all
all: release

.PHONY: test
test: binaries
	build/run_tests.sh $(RELEASE_BRANCH)

.PHONY: clean-repo
clean-repo:
	rm -rf ./kubernetes

.PHONY: clean
clean: clean-repo
	rm -rf ./_output
