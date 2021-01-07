# NestJS for Vizzuality projects

January 2021 - for NestJS 7

This whole documentation should probably become a cookiecutter project based
on the NestJS TypeScript starter, but until then...

## Basic setup

Clone the NestJS TypeScript starter project.

Set up eslint, Husky (if using it - beware of license changes between 4.x and
5.x).

## Config and envvar setup

Configure environment variables. We'll want `POSTGRES_URL` and
`NETWORK_CORS_ORIGINS` to start with. We map these from env vars to `config`
data in `config/custom-environment-variables.json`.

```
{
  "postgres": {
    "url": "POSTGRES_URL"
   },
  "network": {
    "cors": {
        "origins_extra": "NETWORK_CORS_ORIGINS"
    }
  }
}
```

## Database setup

Add db setup: https://docs.nestjs.com/techniques/database.

Typically we'd use PostgreSQL here, so:

```
yarn add @nestjs/typeorm typeorm pg
```

Most likely we'll need `pg-query-stream` at some point so it may make sense to
add this early on, but let's skip it for the moment.

Configure the PostgreSQL connection. We should use the `ormconfig.ts`, and
fetch PostgreSQL URL data via `config`.

For reference: https://typeorm.io/#/using-ormconfig

Using a TypeScript file for the TypeORM configuration allows us to easily add
simple logic to use different settings according to the `NODE_ENV`, e.g.

```
[...]
  ssl: ['staging', 'production'].includes(config.util.getEnv('NODE_ENV')) ? true : false,
[...]
```
