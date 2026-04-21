CONTAINER ID   IMAGE                                       COMMAND                  CREATED          STATUS                    PORTS                                                   NAMES
292b549945df   ubuntu:latest                               "bash -c 'apt-get up…"   5 minutes ago    Up 5 minutes                                                                      app-3tiers_app_1
7ca89422c36b   nginx:latest                                "/docker-entrypoint.…"   5 minutes ago    Up 5 minutes              0.0.0.0:80->80/tcp, [::]:80->80/tcp                     app-3tiers_nginx_1
a7799e7d94ff   mysql:5.7                                   "docker-entrypoint.s…"   5 minutes ago    Up 5 minutes              3306/tcp, 33060/tcp                                     app-3tiers_db_1

# app → nginx ✅
docker exec app-3tiers_app_1 ping -c 2 app-3tiers_nginx_1

# app → db ✅
docker exec app-3tiers_app_1 ping -c 2 app-3tiers_db_1

# nginx → db ❌ (doit échouer)
docker exec app-3tiers_nginx_1 ping -c 2 app-3tiers_db_1


nginx → connecté uniquement à app (frontend)
app → connecté aux deux (nginx et db) — c'est le pont
db → connecté uniquement à app (backend)
nginx ↔ db → complètement isolés dans les deux sens 🚫
