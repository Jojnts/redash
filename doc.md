## Running locally
See https://redash.io/help/open-source/dev-guide/docker for details, but basically:
```sh
# Create Docker services
docker-compose up
# Create (fresh) database
docker-compose run --rm server create_db
```

## Running locally with prod database
First, dump the current redash database from Aptible:
1. Make a snapshot of the database: `aptible db:backup ja-redash`
    1. List backups to get the id (denote it `x`) of the backup just created: `aptible backup:list ja-redash`
    2. Restore into a (temporary) database container with the handle `redash-db-backup`: `aptible backup:restore x --handle redash-db-backup`
    3. Download a dump: `aptible db:dump redash-db-backup`, you'll get a file `redash-db-backup.dump`
    4. Deprovision backup db container: `aptible db:deprovision redash-db-backup`

2. Before building the app with `docker-compose up`, add the dump file path to `.dockerignore` temporarily (or move it out of the project directory).
3. `docker-compose up -d`
4. Import the dump into the running postgres container:
    1. `docker-compose stop server worker`
    2. `docker-compose exec -u postgres postgres dropdb postgres && docker-compose exec -u postgres postgres createdb postgres`
    3. `cat redash-db-backup.dump | docker-compose exec -T -u postgres postgres psql`
    4. `docker-compose up -d`

5. Since we use OAUth on production, it won't be possible (easily at least) to login with your normal user account, therefore, create a new one:
    1. `docker-compose exec server bash`
    2. `./manage.py db upgrade` (if a Redash upgrade requires migrations to be run)
    3. `./manage.py users create admin@jojnts.com admin --admin`

## Upgrading Redash version
1. Add the upstream git remote as `upstream` and merge in desired tag like so: `git merge vX.Y.Z` where `vX.Y.Z` refers to a specific git tag from upstream.
2. Run through the steps above locally with a fresh production dump to make sure everything is fine.
3. Deploy.

## Deploying
We host our own instance of Redash on Aptible: `ja-redash`.
Since Aptible does not support multi-stage Docker builds, we can't use Dockerfile deploy (anymore). Instead we use [Aptible Direct Image Deploy](https://www.aptible.com/documentation/deploy/reference/apps/image/direct-docker-image-deploy/using-aptible-deploy.html),
it works by first building the image ourselves locally (typically done by CI), then pushing it to a Docker registry.
We use TreeScale as our Docker registry.
1. Build the image locally like so `docker build . -t ja-redash:vX` where X should denote the version of Redash you're building.
2. Tag the image `docker tag xxxx repo.treescale.com/axellarsson/ja-redash:vX` where xxx is the id of the recently built Docker image (`docker images | grep ja-redash | grep latest`)
3. Push it to the registry `docker push repo.treescale.com/axellarsson/ja-redash:vX`
4.
```
aptible deploy --app ja-redash \
        --docker-image repo.treescale.com/axellarsson/ja-redash:vX \
        --private-registry-username <tree-scale-username> \
        --private-registry-password <tree-scale-password>
```

5. SSH in to the app and manually run the migrations: `/app/manage.py db upgrade`

### Special case: upgrading from 4.0.0 to 5.0.2
Unfortunately Redash migrations aren't completely forwards compatible - there is a migration: [migrations/versions/969126bd800f_.py](migrations/versions/969126bd800f_.py) that must have the "dashboard.tags" column available. So before running the migration script, manually create that column:
```sql
ALTER TABLE dashboards ADD COLUMN tags text[]
```
Now, `manage db upgrade` should work properly.

