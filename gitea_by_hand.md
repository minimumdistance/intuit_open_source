# Local Gitea Pull Request Demo (Work In Progress)

> NOTE: This document incorporates the cleanup fixes we discovered. The bootstrap/install section is still being validated. Do not assume the compose/app.ini approach is fully working until `gitea admin user create` succeeds.

## 0. Clean Start

```bash
# start from a known location
cd ~

# remove previous demo if present
if [ -d ~/gitea-demo ]; then
  cd ~/gitea-demo

  # stop containers and remove volumes
  docker compose down -v --remove-orphans || true

  cd ~
  rm -rf ~/gitea-demo
fi

# sample repo
rm -rf ~/demo-dev1

# recreate workspace
mkdir ~/gitea-demo
cd ~/gitea-demo

mkdir data config
```

---

## 1. Create Docker Compose

```bash
cat > docker-compose.yml <<'EOF'
services:
  gitea:
    image: gitea/gitea:latest-rootless
    container_name: gitea
    restart: unless-stopped

    ports:
      - "3000:3000"
      - "2222:2222"

    volumes:
      - ./data:/var/lib/gitea
      - ./config:/etc/gitea

    entrypoint:
      - /bin/sh
      - -c
      - |
        mkdir -p /etc/gitea

        cat >/etc/gitea/app.ini <<'APPINI'
        APP_NAME = Local Gitea
        RUN_MODE = prod

        [database]
        DB_TYPE = sqlite3
        PATH = /var/lib/gitea/gitea.db

        [repository]
        ROOT = /var/lib/gitea/git/repositories

        [server]
        DOMAIN = localhost
        ROOT_URL = http://localhost:3000/
        HTTP_PORT = 3000
        SSH_PORT = 2222
        START_SSH_SERVER = true

        [security]
        INSTALL_LOCK = true
        SECRET_KEY = demo-secret
        INTERNAL_TOKEN = demo-token

        [service]
        DISABLE_REGISTRATION = false

        [log]
        MODE = console
        LEVEL = Info
        APPINI

        exec /usr/local/bin/gitea web --config /etc/gitea/app.ini
EOF
```

---

## 2. Start Gitea

Terminal 1:

```bash
docker compose up -d
```

Wait for:

```text
Listen: http://0.0.0.0:3000
```

---

## 3. Verify Install Lock as Gitea Admin

Terminal 2:

```bash
docker compose exec gitea \
grep INSTALL_LOCK /etc/gitea/app.ini
```

Expected:

```text
INSTALL_LOCK = true
```

---

## 4. Create Admin User as Gitea Admin

```bash
docker compose exec gitea \
gitea admin user create \
  --config /etc/gitea/app.ini \
  --username dev1 \
  --password password \
  --email dev1@example.com \
  --must-change-password=false \
  --admin
```

Verify:

```bash
docker compose exec gitea \
gitea admin user list \
  --config /etc/gitea/app.ini
```

---

## 5. Create Contributor User as Gitea Admin

```bash
docker compose exec gitea \
gitea admin user create \
  --config /etc/gitea/app.ini \
  --username dev2 \
  --password password \
  --email dev2@example.com \
  --must-change-password=false
```

---

## 6. Create API Token for dev1 (No UI) as Gitea Admin

```bash
docker compose exec gitea \
gitea admin user generate-access-token \
  --config /etc/gitea/app.ini \
  --username dev1 \
  --token-name automation
```

Save in dev1 terminal:

```bash
export DEV1_TOKEN=<returned-token>
```

---

## 7. Create Repository as dev1

```bash
curl \
  -X POST \
  -H "Authorization: token $DEV1_TOKEN" \
  -H "Content-Type: application/json" \
  http://localhost:3000/api/v1/user/repos \
  -d '{
        "name":"demo",
        "private":false
      }'
```

---

## 8. Clone as dev1

```bash
mkdir ~/demo-dev1
cd ~/demo-dev1

git clone http://localhost:3000/dev1/demo.git
cd demo

git config user.name dev1
git config user.email dev1@example.com
```

Create initial content:

```bash
echo "hello world" > app.txt

git add .
git commit -m "initial commit"
git push
```

---

## 9. Create API Token for dev2 (No UI) as Gitea Admin

```bash
docker compose exec gitea \
gitea admin user generate-access-token \
  --config /etc/gitea/app.ini \
  --username dev2 \
  --token-name automation
```

Save in dev2 terminal:

```bash
export DEV2_TOKEN=<returned-token>
```

---

## 10. Fork Repository as dev2

```bash
curl \
  -X POST \
  -H "Authorization: token $DEV2_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{}' \
  http://localhost:3000/api/v1/repos/dev1/demo/forks
```

---

## 11. Clone Fork as dev2

Terminal 3:

```bash
mkdir ~/demo-dev2
cd ~/demo-dev2

git clone http://localhost:3000/dev2/demo.git
cd demo

git config user.name dev2
git config user.email dev2@example.com
```

---

## 12. Create PR #1 as dev2

```bash
echo "feature from dev2" >> app.txt

git add .
git commit -m "feature 1"
git push
```

Create PR:

```
export PR_NUM=$(
curl -s \
  -X POST \
  -H "Authorization: token $DEV2_TOKEN" \
  -H "Content-Type: application/json" \
  http://localhost:3000/api/v1/repos/dev1/demo/pulls \
  -d '{
        "title":"Feature 1",
        "head":"dev2:main",
        "base":"main"
      }' \
| tee /tmp/pr1.json \
| jq -r '.number'
)

echo "PR_NUM=$PR_NUM"
```

---

## 13. Merge PR #1 as dev1

```
export PR_NUM=... from dev2 terminal
```

```bash
# metadata related to PR
curl -s \
  -H "Authorization: token $DEV1_TOKEN" \
  http://localhost:3000/api/v1/repos/dev1/demo/pulls/$PR_NUM \
| jq
```

```bash
# view diff
curl -s \
  -H "Authorization: token $DEV1_TOKEN" \
  http://localhost:3000/api/v1/repos/dev1/demo/pulls/$PR_NUM.diff
```

```bash
curl \
  -X POST \
  -H "Authorization: token $DEV1_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
        "Do":"merge"
      }' \
  http://localhost:3000/api/v1/repos/dev1/demo/pulls/$PR_NUM/merge
```

```
curl -s \
  -H "Authorization: token $DEV1_TOKEN" \
  http://localhost:3000/api/v1/repos/dev1/demo/pulls/$PR_NUM \
| jq '{number,state,merged,merged_at}'
```

```
cd ~/demo-dev1/demo

git pull

cat app.txt
```

---

## 14. Create PR #2 as dev2

Terminal 3:

```bash
echo "THIS BREAKS EVERYTHING" >> app.txt

git add .
git commit -m "bad change"
git push
```

Create PR:

```bash
export PR_NUM=$(
curl -s \
  -X POST \
  -H "Authorization: token $DEV2_TOKEN" \
  -H "Content-Type: application/json" \
  http://localhost:3000/api/v1/repos/dev1/demo/pulls \
  -d '{
        "title":"Bad Change",
        "head":"dev2:main",
        "base":"main"
      }' \
| tee /tmp/pr2.json \
| jq -r '.number'
)

echo "PR_NUM=$PR_NUM"
```

---

## 15. Reject PR #2 as dev1

```
export PR_NUM=... from dev2 terminal
```

Comment:

```bash
curl \
  -X POST \
  -H "Authorization: token $DEV1_TOKEN" \
  -H "Content-Type: application/json" \
  http://localhost:3000/api/v1/repos/dev1/demo/issues/$PR_NUM/comments \
  -d '{
        "body":"Rejected. Does not meet requirements."
      }'
```

Close:

```bash
curl \
  -X PATCH \
  -H "Authorization: token $DEV1_TOKEN" \
  -H "Content-Type: application/json" \
  http://localhost:3000/api/v1/repos/dev1/demo/pulls/$PR_NUM \
  -d '{
        "state":"closed"
      }'
```

## 16. Confirm PR Rejected as dev2

```
curl -s \
  -H "Authorization: token $DEV2_TOKEN" \
  http://localhost:3000/api/v1/repos/dev1/demo/pulls/$PR_NUM \
| jq '{number,state,merged,closed_at}'
```

---

## 17. Result

PR #1

* Created
* Reviewed
* Merged

PR #2

* Created
* Reviewed
* Rejected
* Closed


## 18. Issues Creation and Resolution

### Create Issue

```
curl \
  -X POST \
  -H "Authorization: token $DEV2_TOKEN" \
  -H "Content-Type: application/json" \
  http://localhost:3000/api/v1/repos/dev1/demo/issues \
  -d '{
        "title":"Bug in parser",
        "body":"Reproduce by..."
      }'
```

### List Issues

```
curl \
  -H "Authorization: token $DEV1_TOKEN" \
  http://localhost:3000/api/v1/repos/dev1/demo/issues
```

### Comment Issue

```
curl \
  -X POST \
  -H "Authorization: token $DEV1_TOKEN" \
  -H "Content-Type: application/json" \
  http://localhost:3000/api/v1/repos/dev1/demo/issues/1/comments \
  -d '{"body":"Investigating"}'
```

### Close Issue

```
curl \
  -X PATCH \
  -H "Authorization: token $DEV1_TOKEN" \
  -H "Content-Type: application/json" \
  http://localhost:3000/api/v1/repos/dev1/demo/issues/1 \
  -d '{"state":"closed"}'
```