###########
# Litestream Builder
###########
FROM golang:alpine3.14 as lt-builder

WORKDIR /usr/src/

RUN apk add --no-cache git make musl-dev gcc

# build litestream
RUN git clone https://github.com/benbjohnson/litestream.git litestream
RUN cd litestream ; go install ./cmd/litestream

RUN cp $GOPATH/bin/litestream /usr/src/lt



###########
# Builder
###########
FROM node:16.17.0-alpine3.15 as builder
WORKDIR /usr/src/app

# install node-gyp dependencies
RUN apk add --no-cache python3 make g++

# Copy application dependency manifests to the container image.
# A wildcard is used to ensure both package.json AND package-lock.json are copied.
# Copying this separately prevents re-running npm ci on every code change.
COPY ./nocodb/package*.json ./
COPY ./nocodb/docker/main.js ./docker/main.js
#COPY ./nocodb/docker/start.sh /usr/src/appEntry/start.sh
COPY ./nocodb/docker/start-litestream.sh /usr/src/appEntry/start.sh
COPY ./nocodb/src/lib/public/css/*.css ./docker/public/css/
COPY ./nocodb/src/lib/public/js/*.js ./docker/public/js/
COPY ./nocodb/src/lib/public/favicon.ico ./docker/public/

# install production dependencies,
# reduce node_module size with modclean & removing sqlite deps,
# package built code into app.tar.gz & add execute permission to start.sh
# COPY for installing with relative-deps
COPY ./nc-lib-gui ../
RUN npm i -g relative-deps \
    && npm ci --omit=dev --quiet \
    && npx relative-deps \
    && npx modclean --patterns="default:*" --ignore="nc-lib-gui/**,dayjs/**,express-status-monitor/**,@azure/msal-node/dist/**" --run  \
    && rm -rf ./node_modules/sqlite3/deps \
    && tar -czf ../appEntry/app.tar.gz ./* \
    && chmod +x /usr/src/appEntry/start.sh \
    && rm -rf ../nc-lib-gui

##########
# Runner
##########
FROM alpine:3.15
WORKDIR /usr/src/app

ENV NC_DOCKER 0.6
ENV NODE_ENV production
ENV PORT 8080
ENV NC_TOOL_DIR=/usr/app/data/

RUN apk --update --no-cache add \
    nodejs \
    tar \
    dumb-init

# Copy litestream binary build
COPY  --from=lt-builder /usr/src/lt /usr/src/appEntry/litestream
# Copy packaged production code & main entry file
COPY --from=builder /usr/src/appEntry/ /usr/src/appEntry/

EXPOSE 8080
ENTRYPOINT ["/usr/bin/dumb-init", "--"]

# Start Nocodb
CMD ["/usr/src/appEntry/start.sh"]
