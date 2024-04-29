# Deploying Backstage

Passo a passo replicável do processo de compilação e execução do Backstage com Docker.

### Instalação do Backstage
No terminal, acesse a pasta na qual se deseja criar o projeto.

1. Crie o projeto com o comando `npx @backstage/create-app@latest --skip-install`
2. Instale as dependências com o comando `yarn install`
3. Pode ser necessário atualizar o node ou yarn para instalar as dependências

### Preparar o projeto para Build
1. Instalar as dependências sem atualizar o lockfile rodando o comando `yarn install --frozen-lockfile`
2. Criar os types do Backstage com o comando `yarn tsc`
3. Buildar a aplicação com o comando `yarn build:backend`

### Ajustando o Dockerfile
1. Abrir no vscode o Dockerfile em `packages > backend > Dockerfile`
2. Altere o código do Dockerfile pelo código abaixo
```
FROM node:18-bookworm-slim

# Install isolate-vm dependencies, these are needed by the @backstage/plugin-scaffolder-backend.
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && \
    apt-get install -y --no-install-recommends python3 g++ build-essential && \
    yarn config set python /usr/bin/python3

# Install sqlite3 dependencies. You can skip this if you don't use sqlite3 in the image,
# in which case you should also move better-sqlite3 to "devDependencies" in package.json.
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && \
    apt-get install -y --no-install-recommends libsqlite3-dev

# From here on we use the least-privileged `node` user to run the backend.
USER node

# This should create the app dir as `node`.
# If it is instead created as `root` then the `tar` command below will
# fail: `can't create directory 'packages/': Permission denied`.
# If this occurs, then ensure BuildKit is enabled (`DOCKER_BUILDKIT=1`)
# so the app dir is correctly created as `node`.
WORKDIR /app

# This switches many Node.js dependencies to production mode.
ENV NODE_ENV development

# Copy repo skeleton first, to avoid unnecessary docker cache invalidation.
# The skeleton contains the package.json of each package in the monorepo,
# and along with yarn.lock and the root package.json, that's enough to run yarn install.
COPY --chown=node:node yarn.lock package.json packages/backend/dist/skeleton.tar.gz ./
RUN tar xzf skeleton.tar.gz && rm skeleton.tar.gz

RUN --mount=type=cache,target=/home/node/.cache/yarn,sharing=locked,uid=1000,gid=1000 \
    yarn install --frozen-lockfile --production --network-timeout 300000

# Then copy the rest of the backend bundle, along with any other files we might want.
COPY --chown=node:node packages/backend/dist/bundle.tar.gz app-config*.yaml ./
RUN tar xzf bundle.tar.gz && rm bundle.tar.gz

CMD ["node", "packages/backend", "--config", "app-config.yaml"]
``` 

### Criar a Imagem do Backstage e rodar o docker
1. Criar a imagem executando o comando `docker image build . -f packages/backend/Dockerfile --tag backstage --no-cache`
2. Criar e executar o container do Backstage com o comando `docker run -it -p 7007:7007 backstage`

Feito todo o processo, acesse http://localhost:7007 para usar o Catálogo de Serviços
![aplicacao rodando](https://github.com/FelipeSaadi/backstage/assets/54749257/661e0d4a-aeb8-4ca9-860f-5e9ed222a3de)

## Imagens da execução do passo a passo

### Instalação do Backstage
![image](https://github.com/FelipeSaadi/backstage/assets/54749257/5391c052-3ff9-4c6c-923f-a8d99b5e4cd1)
![image](https://github.com/FelipeSaadi/backstage/assets/54749257/d74df1b3-97b5-4125-b684-bdc7c7ca75bb)
![yarn-install](https://github.com/FelipeSaadi/backstage/assets/54749257/00a91062-8821-487c-bc38-ec04f997cc46)

### Preparar o projeto para Build 
![instalacao](https://github.com/FelipeSaadi/backstage/assets/54749257/d3a2db07-0795-4233-9e1a-6480616ff7ab)
![image](https://github.com/FelipeSaadi/backstage/assets/54749257/7a2bb3e8-afc2-4640-bf82-8266af415368)
![image](https://github.com/FelipeSaadi/backstage/assets/54749257/8a2ab9f8-cb86-4a07-95d5-8e874ceb81dd)
![image](https://github.com/FelipeSaadi/backstage/assets/54749257/2567b3c2-5ef8-4c26-b4fa-b6b7fd7ecab3)

### Criar a Imagem do Backstage e rodar o docker
![image](https://github.com/FelipeSaadi/backstage/assets/54749257/69ad565f-2ecc-430d-a1a5-e76e548a2875)
![image](https://github.com/FelipeSaadi/backstage/assets/54749257/9ae07962-c661-4370-b167-7b4ea6626b74)
