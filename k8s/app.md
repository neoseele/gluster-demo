## Install app

```sh
kubectl apply -f nm-rails.yaml
```

## Initial app db
```sh
POD=`kubectl get pods -o jsonpath='{range.items[*]}{.metadata.name}{"\n"}{end}' | grep rails | head -n 1`
echo $POD

kubectl exec $POD rake db:migrate
```
## Generate test data

```sh
SERVICE=`kubectl describe service srv-nm-rails | grep Ingress: | awk '{print $3}'`
echo $SERVICE

for i in `seq 1 5`;
do
  curl -d "{ \"post\": {\"name\":\"name-$i\",\"title\":\"title-$i\"} }" -H "Content-Type: application/json" -X POST http://$SERVICE/posts.json
  echo
done

curl http://$SERVICE/posts.json
```

## Clean up

```sh
kubectl delete -f nm-rails.yaml
```
