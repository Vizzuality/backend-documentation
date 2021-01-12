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

```json
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

Remember to create a `test.json` configuration file in the `config` folder (this
will likely be initially an empty JSON document: just `{}`), as this will be
needed when using `node-config` in Jest tests.

## ORM and database connection setup

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

```typescript
[...]
  ssl: ['staging', 'production'].includes(config.util.getEnv('NODE_ENV')) ? true : false,
[...]
```

## OpenAPI plugin

See https://docs.nestjs.com/openapi/introduction.

The example configuration from the NestJS documentation covers most of what we
typically need.

If we want to sync the OpenAPI [*document
version*](http://spec.openapis.org/oas/v3.0.3#fixed-fields-0) with the app
versioning/release flow, we could add this configuration snippet:

```typescript
  const swaggerOptions = new DocumentBuilder()
  [...]
    .setVersion(process.env.npm_package_version || 'development')
  [...]
```

If using a global path prefix for the API (e.g.
`app.setGlobalPrefix('/api/v1');`), this must be set *before* the setup of the
OpenAPI documentation, so that this is reflected automatically in the generated
OpenAPI output.

## CORS setup

Most likely needed in every case. See https://docs.nestjs.com/security/cors

In past projects we have used our own origin handler function, which can be
configured via `node-config`, with defaults included in the app's config and
instance-specific whitelisted origins configured via environment variables: see
the `CorsUtils` module in the Marxan API repository.

@debt This module should be moved to a self-standing package with common
utilities for our NestJS projects.

## Basic security settings via Helmet

It may be a good idea to activate [Helmet](https://www.npmjs.com/package/helmet)
for the NestJS app in `main.ts`: see https://docs.nestjs.com/security/helmet for
quick setup notes.

