# Running locally
See https://redash.io/help/open-source/dev-guide/docker for details, but basically:
```sh
# Create Docker services
docker-compose up
# Create (fresh) database
docker-compose run --rm server create_db
# Build frontend
npm install
npm run build
```

# Running locally with prod database
First, dump the current redash database from Aptible:
1. Make a snapshot of the database: `aptible db:backup ja-redash`
  1. List backups to get the id (denote it `x`) of the backup just created: `aptible backups:list ja-redash`
  2. Restore into a (temporary) database container with the handle `redash-db-backup`: `aptible backup:restore x --handle redash-db-backup`
  3. Download a dump: `aptible db:dump redash-db-backup`, you'll get a file `redash-db-backup.dump`
  4. Deprovision backup db container: `aptible db:deprovision redash-db-backup`

2. Before building the app with `docker-compose up`, add the dump file path to `.dockerignore` temporarily (or move it out of the project directory).
3. `docker-compose up`
4. Import the dump into the running postgres container:
  1. `docker-compose stop server worker`
  2. `docker-compose exec -u postgres postgres dropdb postgres && docker-compose exec -u postgres postgres createdb postgres`
  3. `cat redash-db-backup.dump | docker-compose exec -T -u postgres postgres psql`
  4. `docker-compose up -d`

5. Since we use OAUth on production, it won't be possible (easily at least) to login with your normal user account, therefore, create a new one:
  1. `docker-compose exec server bash`
  2. `./manage.py users create admin@jojnts.com admin --admin`

# Upgrading to latest on production
We host our own instance of Redash on Aptible: `ja-redash`.
1. Add the upstream git remote as `upstream` and merge in desired tag like so: `git merge vX.Y.Z`
2. Run through the steps above locally with a fresh production dump to make sure everything is fine.
3. Push changes to `origin`
4. Push changes to `aptible`: `git push aptible master`
5. SSH in to the app and manually:
  1. Build static assets: `cd app; npm install && npm run build`
  2. Run migration: `/app/manage.py db upgrade`

## Special case: upgrading from 4.0.0 to 5.0.2
Unfortunately Redash migrations aren't completely forwards compatible - there is a migration: [migrations/versions/969126bd800f_.py](migrations/versions/969126bd800f_.py) that must have the "dashboard.tags" column available. So before running the migration script, manually create that column:
```sql
ALTER TABLE dashboards ADD COLUMN tags varchar
```
Now, `manage db upgrade` should work properly.

