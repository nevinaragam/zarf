# Earthfile

ARG CONFIG="config.yaml"
ARG RHEL="false"

# Create the env file template
envfile:
  LOCALLY

  RUN printf '# Generated by earthly +envfile \n\
IB_USER=replace_me \n\
IB_PASS=replace_me \n\
RHEL_USER=replace_me \n\
RHEL_PASS=replace_me \n\
RHEL=false' > .env
  
# Test the deployment with vagrant for RHEL
test-rhel:
  LOCALLY
  RUN vagrant destroy -f && vagrant up --no-color rhel$RHEL && \
      echo -e "\n\n\n\033[1;93m  ✅ BUILD COMPLETE.  To access this environment, run \"vagrant ssh rhel$RHEL\"\n\n\n"

# Test the deployment with vagrant for Ubuntu
test-ubuntu:
  LOCALLY
  RUN vagrant destroy -f && vagrant up --no-color ubuntu && \
      echo -e "\n\n\n\033[1;93m  ✅ BUILD COMPLETE.  To access this environment, run \"vagrant ssh ubuntu\"\n\n\n"

test-destroy:
  LOCALLY
  RUN vagrant destroy -f
  
# Used to load the RHEL7 RPMS
rhel-rpms:
  FROM registry1.dso.mil/ironbank/redhat/ubi/ubi$RHEL
  WORKDIR /rpms

  # Register this machine Red Hat, note that you can only have 16 machines with a developer account
  RUN --secret RHEL_USER=+secrets/RHEL_USER --secret RHEL_PASS=+secrets/RHEL_PASS \
      subscription-manager register --auto-attach --username=$RHEL_USER --password=$RHEL_PASS
  
  # Enable the repo needed for container-selinux
  RUN subscription-manager repos --enable=rhel-$RHEL-server-extras-rpms

  # Resolve and save the rpms locally
  RUN yumdownloader --resolve --destdir=/rpms/ container-selinux

  # Pull the rancher gpg public key 
  RUN curl -fL https://rpm.rancher.io/public.key -o rancher.key

  # Download the K3S SELinux RPM 
  RUN curl -L "https://github.com/k3s-io/k3s-selinux/releases/download/v0.3.stable.0/k3s-selinux-0.3-0.el7.noarch.rpm" -o "/rpms/k3s-selinux.rpm"

  SAVE ARTIFACT /rpms

# Copy the helm 3 binary
helm:
  FROM alpine/helm:3.5.3
  SAVE ARTIFACT /usr/bin/helm

# Copy the yq 4 binary
yq:
  FROM  mikefarah/yq
  SAVE ARTIFACT /usr/bin/yq

# The baseline image with common binaries and $CONFIG
common:
  FROM registry1.dso.mil/ironbank/redhat/ubi/ubi8
  WORKDIR /payload

  RUN yum install -y zstd
  
  COPY +helm/helm /usr/bin
  COPY +yq/yq /usr/bin
  COPY ./cli+build/shift-pack /usr/bin
  COPY $CONFIG .

# Fetch the helm charts specified in $CONFIG 
charts:
  FROM +common

  RUN mkdir charts

  RUN yq e '.charts[] | .name + " " + .url' $CONFIG | \
      while read line ; do echo "repo add $line" | xargs -t helm; done

  RUN yq e '.charts[] | .name + "/" + .name + " -d ./charts --version " + .version' $CONFIG | \
      while read line ; do echo "pull $line" | xargs -t helm; done

  SAVE ARTIFACT charts

# Fetch the k3s version specified in $CONFIG
k3s:
  FROM +common

  RUN curl -fL "https://get.k3s.io" -o "init-k3s.sh"

  RUN K3S_VERSION=$(yq e '.k3s.version' $CONFIG) && \
      curl -fL "https://github.com/k3s-io/k3s/releases/download/$K3S_VERSION/{k3s,k3s-images.txt,sha256sum-amd64.txt}" -o "#1" && \
      sha256sum -c --ignore-missing "sha256sum-amd64.txt"

  SAVE ARTIFACT *

# Fetch k3s images and images specified in $CONFIG
images:
  FROM +common

  COPY +k3s/k3s-images.txt k3s-images.txt

  RUN --secret IB_USER=+secrets/IB_USER --secret IB_PASS=+secrets/IB_PASS \
      k3s_images=$(cat "k3s-images.txt" | tr "\n" " ") && \
      app_images=$(yq e '.images | join(" ")' $CONFIG) && \
      images="$app_images $k3s_images" && \
      echo "Cloning: $images" | tr " " "\n " && \
      shift-pack registry login registry1.dso.mil -u $IB_USER -p $IB_PASS && \
      shift-pack registry pull $images images.tar

  SAVE ARTIFACT images.tar

# Compress all assets in a single tar.zst file
compress: 
  FROM +common

  # Pull in artifacts from other build stages
  COPY +k3s/* bin/
  COPY +charts/charts charts
  COPY +images/images.tar images/images.tar

  # Optional include RHEL rpm build step
  IF [ $RHEL != "false" ]
    COPY +rhel-rpms/rpms rpms
  END

  # Quick housekeeping
  RUN rm -f bin/*.txt && mkdir -p rpms

  # Pull in local resources
  COPY init-manifests manifests

  # Compress the tarball
  RUN tar -cv . | zstd -5 -f - -o /export.tar.zst

  SAVE ARTIFACT /export.tar.zst
  
# Final packaging of the binary/tarball/checksum assets
build:
  FROM +common

  RUN cp /usr/bin/shift-pack .

  # Copy the final compressed tarball for shasum / export
  COPY +compress/export.tar.zst shift-pack.tar.zst

  RUN sha256sum -b shift-pack* > shift-pack.sha256

  RUN ls -lah shift-pack*

  SAVE ARTIFACT shift-pack* AS LOCAL ./build/

#######################################################
##    Temporary location for package update targets   #
#######################################################
# Package all repos specified in $CONFIG
update-repos:
  FROM +common
# Package all images specified in $CONFIG
update-images:
  FROM +common
