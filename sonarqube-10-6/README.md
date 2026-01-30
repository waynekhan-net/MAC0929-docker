# sonarqube-10-6

```text
export POSTGRES_PASSWORD=  # FIXME: Set a non-empty password, lest Postgres not work

# Start the project
docker compose up

# Auth as the "sonar" superuser
docker exec -it sonarqube-10-6-postgres psql -U sonar -d sonarqube
```

## References

* [POSTGRES_HOST_AUTH_METHOD](https://github.com/docker-library/docs/blob/master/postgres/README.md#postgres_password)
* [POSTGRES_PASSWORD](https://github.com/docker-library/docs/blob/master/postgres/README.md#postgres_host_auth_method)
