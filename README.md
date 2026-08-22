# WellNow — Setup & DSA

## 1. Database Setup

```sql
USE wellnow;

CREATE TABLE users (
    username VARCHAR(50) NOT NULL PRIMARY KEY,
    password VARCHAR(255) NOT NULL,
    enabled BOOLEAN NOT NULL
);

CREATE TABLE authorities (
    username VARCHAR(50) NOT NULL,
    authority VARCHAR(50) NOT NULL,

    CONSTRAINT fk_authorities_users
        FOREIGN KEY (username)
        REFERENCES users(username)
);

CREATE UNIQUE INDEX ix_auth_username
    ON authorities (username, authority);
```

---

## 2. Git Commands


### Fetch Latest Changes

```bash
git fetch origin
```

### Pull Latest Changes with Rebase

```bash
git pull origin master --rebase
```

### Push Changes

```bash
git push origin master
```

---

## Create Project Package Structure

Run these commands in the project's `src/main/java` package directory:

```powershell
mkdir api\v1\controller
mkdir api\v1\request
mkdir api\v1\response

mkdir user\service
mkdir user\repository
mkdir user\entity
mkdir user\mapper
mkdir user\specification

mkdir config
mkdir security
mkdir exception
mkdir validation

mkdir common\constant
mkdir common\enums
mkdir common\util
mkdir common\response

mkdir integration\client
mkdir integration\dto
```

## Final Package Structure

```text
src/main/java/
└── <base-package>/
    ├── api/
    │   └── v1/
    │       ├── controller/
    │       ├── request/
    │       └── response/
    │
    ├── user/
    │   ├── service/
    │   ├── repository/
    │   ├── entity/
    │   ├── mapper/
    │   └── specification/
    │
    ├── config/
    ├── security/
    ├── exception/
    ├── validation/
    │
    ├── common/
    │   ├── constant/
    │   ├── enums/
    │   ├── util/
    │   └── response/
    │
    └── integration/
        ├── client/
        └── dto/
```

---


