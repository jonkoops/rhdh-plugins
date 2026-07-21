# Copyright Red Hat, Inc.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

# ── Stage 1: build ────────────────────────────────────────────────────────────
FROM node:22-bookworm-slim AS build

RUN apt-get update && apt-get install -y --no-install-recommends \
    python3 make g++ \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

COPY package.json yarn.lock backstage.json ./
COPY --from=yarn-config .yarnrc.yml ./.yarnrc.yml
COPY --from=yarn-releases . ./.yarn/releases/

COPY tsconfig.json .eslintrc.js .eslintignore ./

COPY packages/ ./packages/
COPY plugins/ ./plugins/
COPY app-config.yaml ./

RUN YARN_ENABLE_IMMUTABLE_INSTALLS=false \
    YARN_CACHE_FOLDER=/root/.yarn/berry/cache \
    yarn install

RUN yarn tsc
RUN yarn workspace app build
RUN yarn workspace backend build

# ── Stage 2: production image ─────────────────────────────────────────────────
FROM node:22-bookworm-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
    python3 make g++ \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

COPY --from=build /app/packages/backend/dist/skeleton.tar.gz ./
RUN tar xzf skeleton.tar.gz && rm skeleton.tar.gz

COPY --from=build /app/package.json /app/yarn.lock /app/.yarnrc.yml ./
COPY --from=build /app/.yarn ./.yarn

RUN YARN_ENABLE_IMMUTABLE_INSTALLS=false \
    YARN_CACHE_FOLDER=/root/.yarn/berry/cache \
    yarn workspaces focus --all --production

RUN yarn workspace backend exec node-gyp rebuild || true

RUN rm -rf "$(yarn cache dir)"

COPY --from=build /app/packages/app/package.json packages/app/package.json
RUN ln -sf ../packages/app node_modules/app

COPY --from=build /app/packages/backend/dist/bundle.tar.gz ./
RUN tar xzf bundle.tar.gz && rm bundle.tar.gz

COPY --from=build /app/plugins/dcm-backend/config.d.ts plugins/dcm-backend/config.d.ts
COPY --from=build /app/packages/app/dist packages/app/dist
COPY app-config.yaml app-config.production.yaml ./

RUN chown -R node:node /app
USER node

EXPOSE 7007

CMD ["node", "packages/backend", \
    "--config", "app-config.yaml", \
    "--config", "app-config.production.yaml"]