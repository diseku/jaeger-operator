include $(shell find tests/e2e -name "Makefile" ! -wholename "tests/e2e/Makefile")

export VERTX_IMG ?= jaegertracing/vertx-create-span:operator-e2e-tests
export ELASTIC_IMG ?= docker.elastic.co/elasticsearch/elasticsearch-oss:6.8.6
export OPERATOR_IMAGE_NEXT ?= ${IMG_PREFIX}/jaeger-operator:next
export ASSERT_IMG ?= ${IMG_PREFIX}/asserts-e2e:$(shell date +%s)

# Use a KIND cluster for the E2E tests
USE_KIND_CLUSTER ?= true
# Skip E2E tests where ES external instance is used
SKIP_ES_EXTERNAL ?= false
export E2E_TESTS_TIMEOUT ?= 330


.PHONY: prepare-e2e-tests
prepare-e2e-tests: kuttl build generate-e2e-files

.PHONY: set-assert-e2e-img-name
set-assert-e2e-img-name:
	$(eval ASSERT_IMG=$(shell ASSERT_IMG=$(ASSERT_IMG) ./tests/build-utils/get-assert-e2e-img.sh))

.PHONY: generate-e2e-files
generate-e2e-files: generate bundle set-image-controller build-assert-job


# Build/pull container images used for the testing ###########################
.PHONY: load-operator-image
load-operator-image: docker kind
ifeq ($(USE_KIND_CLUSTER),true)
	$(ECHO) Loading Jager Operator image in KIND cluster
	$(VECHO)$(KIND) load docker-image $(IMG)
else ifeq ($(JAEGER_OLM),false)
	$(MAKE) push
endif

build-assert-job: Dockerfile.asserts $(shell find tests/assert-jobs -type f)
	$(ECHO) Building E2E asserts image
	$(VECHO)docker build -t $(ASSERT_IMG)  -f Dockerfile.asserts . $(DOCKER_BUILD_OPTIONS)
	$(VECHO)echo $(ASSERT_IMG) > $@


.PHONY: load-assert-job
load-assert-job: build-assert-job set-assert-e2e-img-name
ifeq ($(USE_KIND_CLUSTER),true)
	$(VECHO)./hack/load-kind-image.sh $(ASSERT_IMG)
else
	$(ECHO)Pushing the E2E Test asserts Docker image to the remote repository
	$(VECHO)if [ "$(shell docker manifest inspect $(ASSERT_IMG) 2>/dev/null)" != "" ]; then echo "The image is in the registry. Not pushing"; else docker push $(shell cat build-assert-job); fi;
endif


build-e2e-upgrade-image: Dockerfile $(shell find apis bundle cmd config controllers pkg  -type f)
	@# Instead of calling the "docker" make target as a dependency, it is needed to invoke it
	@# as a target step. The reason is: it is needed to use "JAEGER_VERSION" and overwrite it. This causes
	@# an error in Make
	$(VECHO)$(MAKE) docker JAEGER_VERSION=$(shell .ci/get_test_upgrade_version.sh ${JAEGER_VERSION}) IMG="$(OPERATOR_IMAGE_NEXT)"
	@# Create a file with the name of the target to help make to calculate the timestamp
	$(VECHO)touch $@


.PHONY: load-e2e-upgrade-image
load-e2e-upgrade-image: build-e2e-upgrade-image
ifeq ($(USE_KIND_CLUSTER),true)
	$(VECHO)./hack/load-kind-image.sh $(OPERATOR_IMAGE_NEXT)
endif

.PHONY: load-elasticsarch-image
load-elasticsarch-image:
ifneq ($(SKIP_ES_EXTERNAL),true)
ifeq ($(USE_KIND_CLUSTER),true)
	$(VECHO)./hack/pull-docker-image.sh $(ELASTIC_IMG)
	$(VECHO)./hack/load-kind-image.sh $(ELASTIC_IMG)
else
	$(ECHO) The Elasticsearch Docker image will be pulled from the registry by the cluster
endif
else
	$(ECHO) Elasticsearch image not loaded because SKIP_ES_EXTERNAL is true
endif
# End build/pull container images used for the testing #######################

.PHONY: run-e2e-tests
run-e2e-tests: list-test-suites
	./hack/run-e2e-tests.sh $(TEST_SUITES)


.PHONY: render-e2e-tests
render-e2e-tests: generate-e2e-files list-test-suites
	$(VECHO) for suite in $(TEST_SUITES); do \
		$(MAKE) render-e2e-tests-$$suite ; \
	done

.PHONY: list-test-suites
list-test-suites:
	$(eval TEST_SUITES := $(shell find tests/e2e -name Makefile | grep  -E -o '(\w|\-)*' | grep -v 'test' | grep -v 'Makefile' | grep -v 'e2e'))

.PHONE: e2e-test-suites
e2e-test-suites: list-test-suites
	@echo "Test suites to run"
	@for test_suite in $(TEST_SUITES) ; do \
		echo -e "\t $$test_suite" ; \
	done

	@echo "You can run a test suite with make run-e2e-tests-<suite name>. E.g: make run-e2e-tests-elasticsearch"

run-suite-tests: start-kind prepare-e2e-tests load-operator-image
	KAFKA_OLM=$(KAFKA_OLM) ./hack/run-e2e-test-suite.sh $(TEST_SUITE_NAME) $(USE_KIND_CLUSTER) $(JAEGER_OLM)
