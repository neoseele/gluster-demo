```sh
$ kubectl apply -f pvc.yaml
$ kubectl apply -f deployment.yaml
$ kubectl apply -f service.yaml

$ kubectl get pvc
NAME            STATUS    VOLUME                                     CAPACITY   ACCESSMODES   STORAGECLASS     AGE
nm-rails-data   Bound     pvc-cedfd662-bb15-11e7-8417-42010a800ffa   2Gi        RWX           gluster-heketi   21m

$ kubectl get pods
NAME                        READY     STATUS    RESTARTS   AGE
glusterfs-2z09g             1/1       Running   0          2h
glusterfs-6g1lg             1/1       Running   0          2h
glusterfs-dds91             1/1       Running   0          2h
heketi-832879333-qk7f1      1/1       Running   0          1h
nm-rails-1905005039-mmtd5   1/1       Running   0          1m
nm-rails-1905005039-sb4fj   1/1       Running   0          1m
nm-toolbox                  1/1       Running   0          1h

$ kubectl exec nm-rails-1905005039-mmtd5 rake db:migrate
D, [2017-10-27T12:54:52.104791 #7] DEBUG -- :    (43.2ms)  CREATE TABLE "schema_migrations" ("version" varchar NOT NULL)
D, [2017-10-27T12:54:52.105259 #7] DEBUG -- :    (0.1ms)  select sqlite_version(*)
D, [2017-10-27T12:54:52.167911 #7] DEBUG -- :    (48.9ms)  CREATE UNIQUE INDEX "unique_schema_migrations" ON "schema_migrations" ("version")
D, [2017-10-27T12:54:52.183318 #7] DEBUG -- :   ActiveRecord::SchemaMigration Load (7.3ms)  SELECT "schema_migrations".* FROM "schema_migrations"
I, [2017-10-27T12:54:52.194196 #7]  INFO -- : Migrating to CreatePosts (20171014122211)
D, [2017-10-27T12:54:52.194836 #7] DEBUG -- :    (0.1ms)  begin transaction
== 20171014122211 CreatePosts: migrating ======================================
-- create_table(:posts)
D, [2017-10-27T12:54:52.206797 #7] DEBUG -- :    (11.0ms)  CREATE TABLE "posts" ("id" INTEGER PRIMARY KEY AUTOINCREMENT NOT NULL, "name" varchar, "title" varchar, "content" text, "created_at" datetime NOT NULL, "updated_at" datetime NOT NULL)
   -> 0.0119s
== 20171014122211 CreatePosts: migrated (0.0120s) =============================

D, [2017-10-27T12:54:52.214095 #7] DEBUG -- :   SQL (0.7ms)  INSERT INTO "schema_migrations" ("version") VALUES (?)  [["version", "20171014122211"]]
D, [2017-10-27T12:54:52.260206 #7] DEBUG -- :    (45.7ms)  commit transaction
D, [2017-10-27T12:54:52.279929 #7] DEBUG -- :   ActiveRecord::SchemaMigration Load (6.6ms)  SELECT "schema_migrations".* FROM "schema_migrations"

$ kubectl get service
NAME                              CLUSTER-IP      EXTERNAL-IP      PORT(S)           AGE
glusterfs-dynamic-nm-rails-data   10.47.244.99    <none>           1/TCP             22m
heketi                            10.47.250.102   <none>           8080/TCP          2h
heketi-ext                        10.47.241.17    35.192.98.226    18080:30924/TCP   25m
heketi-storage-endpoints          10.47.249.61    <none>           1/TCP             2h
kubernetes                        10.47.240.1     <none>           443/TCP           3h
nm-rails                          10.47.243.111   35.202.217.201   80:32418/TCP      20m

SERVICE=`kubectl describe service nm-rails | grep Ingress: | awk '{print $3}'`

curl http://$SERVICE/posts.json

curl -d '{ "post": {"name":"123","title":"123"} }' -H "Content-Type: application/json" -X POST http://$SERVICE/posts.json
```
