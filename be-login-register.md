# Backend

1. create Makefile

```Makefile
compose-up:
	docker compose up -d --build

compose-down:
	docker compose down

execute-db:
	npx prisma migrate dev --name dev

create-db:
	docker run --name pg-youtube \
	-p 6000:5432 \
	-e PGDATA=/var/lib/postgresql/data/pgdata \
	-e POSTGRES_USER=db-youtube \
	-e POSTGRES_PASSWORD=db-youtube \
	-v data-db:/var/lib/postgresql/data \
	--net youtube-net \
	-d postgres:alpine
```

2. create data-db folder
3. make create-db
4. npm init -y
5. npm i --save @fastify/autoload @fastify/cors @prisma/client dotenv fastify fastify-plugin nodemon prisma
6. modify package.json

```json
{
  "name": "backend",
  "module": "index.js",
  "type": "module",
  "scripts": {
    "dev": "nodemon index.js"
  },
  "dependencies": {
    "@fastify/autoload": "^5.7.1",
    "@fastify/cors": "^8.3.0",
    "@prisma/client": "^5.3.1",
    "dotenv": "^16.3.1",
    "fastify": "^4.23.2",
    "fastify-plugin": "^4.5.1",
    "nodemon": "^3.0.1",
    "prisma": "^5.3.1"
  }
}
```

7. create prisma folder
8. create prisma/migrations folder
9. create schema.prisma

```ts
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        String   @id @default(uuid())
  email     String   @unique @db.VarChar(255)
  name      String   @db.VarChar(255)
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

10. create .env

```sh
APP_HOST=0.0.0.0
APP_PORT=5020
APP_ENV=development

HOST_JWT=http://localhost:5000
HOST_GOOGLE_API=https://www.googleapis.com

DATABASE_URL=postgresql://db-youtube:db-youtube@0.0.0.0:6000/db-youtube?schema=public
```

11. create common.config.js

```js
import dotenv from "dotenv";

dotenv.config();

export const appHost = process.env.APP_HOST;
export const appPort = process.env.APP_PORT;
export const appEnv = process.env.APP_ENV;
export const hostJwt = process.env.HOST_JWT;
```

12. make execute-db
13. create error folder
14. create AppError.js

```js
class AppError extends Error {
  constructor(message, statusCode) {
    super(message);
    this.statusCode = statusCode;
    this.isOperational = true; // Indicates that this is a known, operational error
    Error.captureStackTrace(this, this.constructor);
  }
}

export default AppError;
```

14. create plugins folder
15. create prisma.plugin.js

```js
import { PrismaClient } from "@prisma/client";
import fp from "fastify-plugin";

export default fp(async (app, options) => {
  const prisma = new PrismaClient();
  await prisma.$connect();
  app.decorate("prisma", prisma);
  app.addHook("onClose", async (app) => {
    await app.prisma.$disconnect();
  });
});
```

16. create jwt.plugin.js

```js
import fp from "fastify-plugin";
import { hostJwt } from "../configs/common.config.js";
import AppError from "../error/AppError.js";

export default fp(async (app, options) => {
  app.decorate("token", async (data) => {
    const response = await fetch(`${hostJwt}/token`, {
      headers: {
        "Content-Type": "application/json",
      },
      method: "POST",
      body: JSON.stringify(data),
    });
    if (!response.ok)
      throw new AppError("Couldn't create token: ", response.status);
    return await response.json();
  });

  app.decorate("validate", async (token) => {
    const response = await fetch(`${hostJwt}/validate`, {
      headers: {
        "Content-Type": "application/json",
      },
      method: "POST",
      body: JSON.stringify({ token }),
    });
    if (!response.ok) throw new AppError("Invalid token: ", response.status);
    return await response.json();
  });

  app.decorate("refresh", async (token) => {
    const response = await fetch(`${hostJwt}/refresh`, {
      headers: {
        "Content-Type": "application/json",
      },
      method: "POST",
      body: JSON.stringify({ token }),
    });
    if (!response.ok) throw new AppError("Invalid token: ", response.status);
    return await response.json();
  });
});
```

17. create googleapi.plugin.js

```js
import fp from "fastify-plugin";
import AppError from "../error/AppError.js";

export default fp(async (app, options) => {
  app.decorate("gmailUserInfo", async (accessToken) => {
    const response = await fetch(
      `${process.env.HOST_GOOGLE_API}/oauth2/v1/userinfo?alt=json`,
      {
        headers: {
          Authorization: `Bearer ${accessToken}`,
        },
      }
    );
    if (!response.ok) throw new AppError("Can't access google api OAuth", 400);
    return await response.json();
  });
});
```

18. create users.plugin.js

```js
import fp from "fastify-plugin";
import AppError from "../error/AppError.js";

export default fp(async (app, options) => {
  app.decorate("getUserByEmail", async (email) => {
    const user = await app.prisma.user.findUnique({
      where: { email },
    });
    if (user === null)
      throw new AppError(
        `Can't find user by this email address: ${email}`,
        404
      );
    return user;
  });

  app.decorate("registerUser", async (userData) => {
    try {
      return await app.prisma.user.create({
        data: userData,
      });
    } catch (e) {
      throw new AppError(
        `Can't register this email address: ${userData.email}`,
        400
      );
    }
  });
});
```

19. create handlers folder
20. create register.handler.js

```js
export async function registerByGmail(req, res) {
  try {
    const responseJson = await this.gmailUserInfo(
      req.headers.authorization.split(" ")[1]
    );
    const { email, name, id } = responseJson;
    await this.registerUser({ email, name, id });
    const user = await this.getUserByEmail(email);
    return {
      message: `Your email ${email} has been registered`,
      data: await this.token(user),
    };
  } catch (err) {
    return res.code(err.statusCode).send(err);
  }
}
```

21. create login.handler.js

```js
export async function loginByGmail(req, res) {
  try {
    const responseJson = await this.gmailUserInfo(
      req.headers.authorization.split(" ")[1]
    );
    const { email } = responseJson;
    const user = await this.getUserByEmail(email);
    return res.status(200).send({
      message: `User can be found by ${email}`,
      data: await this.token(user),
    });
  } catch (err) {
    return res.code(err.statusCode).send(err);
  }
}
```

22. create routes folder
23. create auth.routes.js

```js
import { loginByGmail } from "../handlers/login.handler.js";
import { registerByGmail } from "../handlers/register.handler.js";

const routes = async (app, options) => {
  app.route({
    method: "GET",
    url: "/login/gmail",
    handler: loginByGmail,
  });

  app.route({
    method: "GET",
    url: "/register/gmail",
    handler: registerByGmail,
  });
};

export default routes;
```

24. create index.js

```js
import path from "path";
import { appHost, appPort, appEnv } from "./configs/common.config.js";
import { fileURLToPath } from "url";

import Server from "fastify";
const app = new Server({ logger: appEnv === "development" ? true : false });

const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename);

// register cors plugin
import cors from "@fastify/cors";
app.register(cors);
// TODO list url & method yg di allow

// autoload all custom plugins
import autoLoad from "@fastify/autoload";
app.register(autoLoad, {
  dir: path.join(__dirname, "plugins"),
});

// autoload all routes
app.register(autoLoad, {
  dir: path.join(__dirname, "routes"),
});

app.listen({ host: appHost, port: appPort }, (err, _) => {
  if (err) {
    console.error(err);
  }
});
```
