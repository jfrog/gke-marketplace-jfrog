ARG MARKETPLACE_TOOLS_TAG
FROM marketplace.gcr.io/google/debian9 AS build

RUN apt-get update \
    && apt-get install -y --no-install-recommends gettext

ADD chart/unified-mp /tmp/chart

ARG TAG

RUN cat /tmp/chart/values.yaml \
    | env -i "TAG=$TAG" envsubst \
    > /tmp/chart/values.yaml.new \
    && mv /tmp/chart/values.yaml.new /tmp/chart/values.yaml \
    && cd /tmp && tar -czvf /tmp/artifactory-mp.tar.gz chart

RUN cat /tmp/chart/Chart.yaml \
    | env -i "TAG=$TAG" envsubst \
    > /tmp/chart/Chart.yaml.new \
    && mv /tmp/chart/Chart.yaml.new /tmp/chart/Chart.yaml \
    && cd /tmp && tar -czvf /tmp/artifactory-mp.tar.gz chart

RUN cd /tmp && tar -czvf /tmp/artifactory-mp.tar.gz chart

ADD apptest/deployer/artifactory-ha /tmp/test/chart
RUN cd /tmp/test \
    && tar -czvf /tmp/test/artifactory-mp.tar.gz chart/

ADD schema.yaml /tmp/schema.yaml

ARG REGISTRY
ARG TAG

RUN cat /tmp/schema.yaml \
    | env -i "REGISTRY=$REGISTRY" "TAG=$TAG" envsubst \
    > /tmp/schema.yaml.new \
    && mv /tmp/schema.yaml.new /tmp/schema.yaml

ADD apptest/deployer/schema.yaml /tmp/apptest/schema.yaml
RUN cat /tmp/apptest/schema.yaml \
    | env -i "REGISTRY=$REGISTRY" "TAG=$TAG" envsubst \
    > /tmp/apptest/schema.yaml.new \
    && mv /tmp/apptest/schema.yaml.new /tmp/apptest/schema.yaml

FROM gcr.io/cloud-marketplace-tools/k8s/deployer_helm
COPY --from=build /tmp/artifactory-mp.tar.gz /data/chart/
COPY --from=build /tmp/test/artifactory-mp.tar.gz /data-test/chart/
COPY --from=build /tmp/apptest/schema.yaml /data-test/
COPY --from=build /tmp/schema.yaml /data/

ENV WAIT_FOR_READY_TIMEOUT 3600
ENV TESTER_TIMEOUT 3600