VERSION 0.6
FROM alpine

ARG BASE_IMAGE=quay.io/kairos/core-opensuse:v1.1.4
ARG IMAGE_REPOSITORY=quay.io/kairos

ARG LUET_VERSION=0.33.0
ARG GOLINT_VERSION=v1.46.2
ARG GOLANG_VERSION=1.18

ARG RKE2_VERSION=latest
ARG BASE_IMAGE_NAME=$(echo $BASE_IMAGE | grep -o [^/]*: | rev | cut -c2- | rev)
ARG BASE_IMAGE_TAG=$(echo $BASE_IMAGE | grep -o :.* | cut -c2-)
ARG RKE2_VERSION_TAG=$(echo $RKE2_VERSION | sed s/+/-/)

build-cosign:
    FROM gcr.io/projectsigstore/cosign:v1.9.0
    SAVE ARTIFACT /ko-app/cosign cosign

go-deps:
    FROM golang:$GOLANG_VERSION
    WORKDIR /build
    COPY go.mod go.sum ./
    RUN go mod download
    RUN apt-get update && apt-get install -y upx
    SAVE ARTIFACT go.mod AS LOCAL go.mod
    SAVE ARTIFACT go.sum AS LOCAL go.sum

BUILD_GOLANG:
    COMMAND
    WORKDIR /build
    COPY . ./
    ARG BIN
    ARG SRC

    RUN go build -ldflags "-s -w" -o ${BIN} ./${SRC} && upx ${BIN}
    SAVE ARTIFACT ${BIN} ${BIN} AS LOCAL build/${BIN}

VERSION:
    COMMAND
    FROM alpine
    RUN apk add git

    COPY . ./

    RUN echo $(git describe --exact-match --tags || echo "v0.0.0-$(git log --oneline -n 1 | cut -d" " -f1)") > VERSION

    SAVE ARTIFACT VERSION VERSION

luet:
    FROM quay.io/luet/base:$LUET_VERSION
    SAVE ARTIFACT /usr/bin/luet /luet

build-provider:
    FROM +go-deps
    DO +BUILD_GOLANG --BIN=agent-provider-rke2 --SRC=main.go

lint:
    FROM golang:$GOLANG_VERSION
    RUN wget -O- -nv https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh | sh -s $GOLINT_VERSION
    WORKDIR /build
    COPY . .
    RUN golangci-lint run

docker:
    DO +VERSION
    ARG VERSION=$(cat VERSION)

    FROM $BASE_IMAGE
    COPY +luet/luet /usr/bin/luet
    IF [ "$RKE2_VERSION" = "latest" ]
    ELSE
        ENV INSTALL_RKE2_VERSION=${RKE2_VERSION}
    END

    COPY install_rke2.sh .

    ENV INSTALL_RKE2_METHOD="tar"
    ENV INSTALL_RKE2_SKIP_RELOAD="true"
    ENV INSTALL_RKE2_TAR_PREFIX="/usr"
    RUN ./install_rke2.sh && rm install_rke2.sh
    COPY +build-provider/agent-provider-rke2 /system/providers/agent-provider-rke2

    ENV OS_ID=${BASE_IMAGE_NAME}-rke2
    ENV OS_NAME=$OS_ID:${BASE_IMAGE_TAG}
    ENV OS_REPO=${IMAGE_REPOSITORY}
    ENV OS_VERSION=${RKE2_VERSION_TAG}_${VERSION}
    ENV OS_LABEL=${BASE_IMAGE_TAG}_${RKE2_VERSION_TAG}_${VERSION}
    RUN envsubst >/etc/os-release </usr/lib/os-release.tmpl

    SAVE IMAGE --push $IMAGE_REPOSITORY/${BASE_IMAGE_NAME}-rke2:${RKE2_VERSION_TAG}
    SAVE IMAGE --push $IMAGE_REPOSITORY/${BASE_IMAGE_NAME}-rke2:${RKE2_VERSION_TAG}_${VERSION}

cosign:
    ARG --required ACTIONS_ID_TOKEN_REQUEST_TOKEN
    ARG --required ACTIONS_ID_TOKEN_REQUEST_URL

    ARG --required REGISTRY
    ARG --required REGISTRY_USER
    ARG --required REGISTRY_PASSWORD

    DO +VERSION
    ARG VERSION=$(cat VERSION)

    FROM docker

    ENV ACTIONS_ID_TOKEN_REQUEST_TOKEN=${ACTIONS_ID_TOKEN_REQUEST_TOKEN}
    ENV ACTIONS_ID_TOKEN_REQUEST_URL=${ACTIONS_ID_TOKEN_REQUEST_URL}

    ENV REGISTRY=${REGISTRY}
    ENV REGISTRY_USER=${REGISTRY_USER}
    ENV REGISTRY_PASSWORD=${REGISTRY_PASSWORD}

    ENV COSIGN_EXPERIMENTAL=1
    COPY +build-cosign/cosign /usr/local/bin/

    RUN echo $REGISTRY_PASSWORD | docker login -u $REGISTRY_USER --password-stdin $REGISTRY

    RUN cosign sign $IMAGE_REPOSITORY/${BASE_IMAGE_NAME}-rke2:${RKE2_VERSION_TAG}
    RUN cosign sign $IMAGE_REPOSITORY/${BASE_IMAGE_NAME}-rke2:${RKE2_VERSION_TAG}_${VERSION}
