# Sample Python Application hosted in Docker container

## Build Docker image

```
docker build -t hongjee/sample-python-in-docker .
```

## Test in local Docker container

```
docker run --rm -p 8000:8000 -v $PWD/logs:/tmp/sample-app hongjee/sample-python-in-docker
```

## Test the web application

```
http://localhost:8000/
```

## Clean up

```
sudo rm -r logs
docker rmi hongjee/sample-python-in-docker
```

