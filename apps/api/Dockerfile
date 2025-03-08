FROM node:20-alpine3.19 as dev_base
RUN apk add g++ make py3-pip

# Remove Enterprise dependencies
ARG BULL_MQ_PRO_TOKEN
ENV NX_DAEMON=false

RUN npm i pm2 -g
RUN npm --no-update-notifier --no-fund --global install pnpm@8.9.0
RUN pnpm --version

USER 1000
WORKDIR /usr/src/app

# ------- DEV BUILD ----------
FROM dev_base AS dev
ARG PACKAGE_PATH

COPY --chown=1000:1000 ./meta .
COPY --chown=1000:1000 ./deps .
COPY --chown=1000:1000 ./pkg .

# Skip Enterprise Edition
RUN echo "Building Community Edition Only"

# Install only required dependencies
RUN --mount=type=cache,id=pnpm-store-api,target=/root/.pnpm-store\
 pnpm install --filter "musama619" --filter "{${PACKAGE_PATH}}..."\
 --frozen-lockfile \
 --unsafe-perm

RUN NODE_ENV=production NX_DAEMON=false pnpm build:api

WORKDIR /usr/src/app/apps/api

RUN cp src/.example.env dist/src/.env
RUN cp src/.env.test dist/src/.env.test
RUN cp src/.env.development dist/src/.env.development
RUN cp src/.env.production dist/src/.env.production

WORKDIR /usr/src/app

# ------- ASSETS BUILD ----------
FROM dev AS assets

WORKDIR /usr/src/app

# Remove all dependencies so later we can only install prod dependencies
RUN rm -rf node_modules && pnpm recursive exec -- rm -rf ./src ./node_modules

# ------- PRODUCTION BUILD ----------
FROM dev_base AS prod

ARG PACKAGE_PATH
ENV CI=true

WORKDIR /usr/src/app

COPY --chown=1000:1000 ./meta .

# Get the build artifacts that only include dist folders
COPY --chown=1000:1000 --from=assets /usr/src/app .

RUN --mount=type=cache,id=pnpm-store-api,target=/root/.pnpm-store\
 pnpm install --filter "{${PACKAGE_PATH}}..." \
 --frozen-lockfile \
 --unsafe-perm

ENV NEW_RELIC_NO_CONFIG_FILE=true

WORKDIR /usr/src/app/apps/api
CMD [ "pm2-runtime", "start", "dist/main.js" ]
