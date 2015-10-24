# Create rpm packges

## Create mysql release repo package

Just for example when create mysql release repo rpm for EL7:
```
$ cd mysql/el7
$ fpm -s dir -t rpm --name mysql-community-release -a noarch -v el7 -f --prefix / etc
```
