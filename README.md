# NodeFirebaseFunctionsBible
Este documento tiene como objetivo describir el conocimiento sobre Mode, Express y Firebase Cloud Functions.

## Estructura de carpetas

```
  /
  ├── functions/                
      ├── actions/   	    
      ├── config/  
      ├── controllers/    
      ├── middlewares/      
      ├── index.js		--> Punto de entrada
```

## Nodemon

* [nodemon](https://nodemon.io)
* [GitHub - remy/nodemon: Monitor for any changes in your...](https://github.com/remy/nodemon)

La librería Nodemon nos proporciona algo parecido a un Hot Reloading. Nos facilita el desarrollo en local, ya que reinicia el servidor node local al guardar cualquier cambio. 

Hay instalarlo de forma global

```shell
$ npm install -g nodemon
```

Luego añadimos en el package.json del proyecto un nuevo script para arrancar el servidor en local con nodemon.

```json
"scripts": {
    "local": "nodemon ./index.js -l"
  },
```

Le pasamos el parámetro -l para poder detectar si estamos en desarrollo local o en remoto. Para ello, podemos añadir en el index.js la variable  __DEV__ para saber si estamos en desarrollo local.

```javascript
// src/index.js

const minimist = require("minimist");
const __DEV__ = minimist(process.argv.slice(2)).l;
//__DEV__ es true si hemos ejecutado con nodemon en local (-l)

// ...
// Configuración
// Middlewares
// Endpoints (express)
// ...

if (__DEV__) {
  app.listen(3001, () => console.log("app on http://localhost:3001"));
} else {
  exports.app = functions.https.onRequest(app);
}
```

Para arrancar nuestro entorno de desarrollo

```shell
$ cd /functions
$ yarn local
```

## Middlewares

Express

```javascript
const express = require("express");
const app = express();
app.use(express.json());
```

Cookie Parser

```javascript
const cookieParser = require("cookie-parser")();
app.use(cookieParser);
```

Cors

```javascript
const cors = require("cors")({ origin: true });
app.use(cors);
```

Body Parser

```javascript
const bodyParser = require("body-parser");
app.use(bodyParser.urlencoded({ extended: true }));
app.use(bodyParser.json());
```

Logger

```javascript
const loggerMiddleware = require("./src/middleware/logger");
app.use(loggerMiddleware());
```

```javascript
// src/middleware/logger.js

var rs = require("random-strings");

module.exports = () => {
  return (req, res, next) => {
    var hash = rs.hex(6);
    var ip = req.headers["x-forwarded-for"] || req.connection.remoteAddress;
    var start = Date.now();
    var method = req.method;
    var url = req.url;
    var referer = req.headers.referer || "";
    var ua = req.headers["user-agent"];

    console.log(hash, method, url, referer, ua, ip);

    res.on("finish", () => {
      var code = "(" + res._header ? String(res.statusCode) : String(-1) + ")";
      var duration = Date.now() - start + "ms";
      if (res.statusCode < 300) {
        console.log(code, method, url, duration);
      } else if (res.statusCode < 500) {
        console.warn(code, method, url, duration);
      } else {
        console.error(code, method, url, duration);
      }
    });

    next();
  };
};
```

Auth con Firebase

```javascript
const authMiddleware = require("./src/middleware/auth");
app.use(authMiddleware);
```

```javascript
// src/middleware/auth.js

const admin = require("firebase-admin");

// Express middleware that validates Firebase ID Tokens passed in the Authorization HTTP header.
// The Firebase ID token needs to be passed as a Bearer token in the Authorization HTTP header like this:
// `Authorization: Bearer <Firebase ID Token>`.
// when decoded successfully, the ID Token content will be added as `req.user`.

module.exports = async (req, res, next) => {
  if (
    (!req.headers.authorization ||
      !req.headers.authorization.startsWith("Bearer ")) &&
    !(req.cookies && req.cookies.__session)
  ) {
    res.status(403).send("Unauthorized");
    return;
  }

  let idToken;
  if (
    req.headers.authorization &&
    req.headers.authorization.startsWith("Bearer ")
  ) {
    // Read the ID Token from the Authorization header.
    idToken = req.headers.authorization.split("Bearer ")[1];
  } else if (req.cookies) {
    // Read the ID Token from cookie.
    idToken = req.cookies.__session;
  } else {
    // No cookie
    res.status(403).send("Unauthorized");
    return;
  }

  try {
    req.user = await admin.auth().verifyIdToken(idToken);
	next();
  } catch (error) {
    res.status(403).send("Unauthorized");
  }
};
```

## Desplegar en Firebase

```shell
$ cd /functions
$ firebase deploy
```
